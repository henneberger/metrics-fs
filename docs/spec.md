# metricfs Specification (v0.2)

## 1. Problem statement

You have JSONL metrics on disk and authorization policy in SpiceDB. You want
AI/users to use standard shell tooling (`cat`, `grep`, `awk`, `jq`) while
enforcing row-level visibility.

Required behavior:

- Mount a virtual filesystem over source metrics.
- Unauthorized rows are omitted (not redacted).
- Policy changes become visible quickly on new file opens.

## 2. Goals

- Transparent file/CLI workflow.
- Deterministic row-level authorization decisions.
- Fast policy propagation from SpiceDB.
- Support heterogeneous metric formats through per-directory mapping files.

## 3. Non-goals (MVP)

- Multi-writer filesystem semantics.
- SQL engine/query planner.
- High-throughput scan optimization as a primary objective.

## 4. Architecture

## 4.1 Components

1. Mount layer (FUSE):
- Exposes source files under mount path.
- Intercepts `open/read/readdir/getattr`.
- Filters `.jsonl` reads only; non-JSONL is read-only passthrough.

2. Mapper resolver:
- Discovers `.metricfs-map.yaml` from nearest ancestor directory.
- Applies optional `extends` inheritance.
- Produces an immutable compiled rule set per directory.

3. File indexer:
- Builds sidecar index keyed by `(inode,size,mtime_ns,mapper_hash)`.
- Stores per line:
  - byte `start/end` offsets
  - `decision` mode (`any|all`)
  - candidate list `[(object_type, object_id, permission), ...]`
  - parse/extraction status
- Supports multi-candidate rows (required for OpenLineage).

4. Permission snapshot service:
- Maintains allowed object-id sets keyed by `(object_type, permission)`.
- Seeds from SpiceDB `LookupResources`.
- Maintains freshness with `Watch` + periodic reconciliation.
- Publishes immutable snapshots and monotonic `policy_epoch`.

5. View evaluator:
- On file open, evaluates each indexed line against current snapshot.
- Produces a visible-line bitmap and `visible_segments` for virtual reads.
- Caches by `(path_fingerprint, subject, policy_epoch)`.

## 4.2 Read path

1. `open("/mnt/.../file.jsonl")`
2. Resolve real source path and mapper rules.
3. Build or load index.
4. Load current permission snapshot for subject.
5. Evaluate line visibility and construct visible segments.
6. Serve `read(offset,len)` by mapping virtual offsets to backing segments.

Consistency:

- Snapshot-at-open is required for MVP.
- Existing file handles keep open-time snapshot.
- New opens observe new policy epochs.

## 4.3 Row authorization algorithm (normative)

For each line:

1. Parse JSON row.
2. Apply matching mapper rule to emit candidate objects.
3. If candidate field extraction fails:
  - `missing_resource_key=deny`: deny line.
  - `missing_resource_key=ignore`: drop candidate and continue.
4. Dedupe candidates.
5. If zero candidates remain: deny line.
6. Evaluate candidate allow set membership:
  - `decision=any`: allow line if at least one candidate is allowed.
  - `decision=all`: allow line only if every candidate is allowed.

## 5. Mapping model and DSL

Responsibilities:

- `metricfs`: parsing + canonicalization to object keys.
- SpiceDB: relationship/transitive authorization semantics.

## 5.1 Mapper discovery

- File name is configurable (`.metricfs-map.yaml` default).
- Resolver mode is `nearest_ancestor` (MVP only).
- `extends` may reference a parent mapper file.
- Effective rule order is: local file rules first, then inherited rules.
- `extends` cycles are invalid configuration.
- Rule order is deterministic; first matching rule wins.
- If no mapper file or no matching rule is found:
  - behavior controlled by `--missing-mapper` (default `deny`).

## 5.2 Rule schema (normative)

Common rule fields:

- `match.glob` (required)
- `decision` (`any|all`, default `any`)
- `missing_resource_key` (`deny|ignore`, default `deny`)
- `mapper` (required)

Mapper kinds:

1. `json_pointer` (single candidate)
- Rule-level `object_type` and `permission` are required.
- `mapper.pointer` must be a root pointer (`/path`).
- `mapper.canonical_template` typically uses `{value}`.

2. `multi_extract` (multiple candidates)
- `mapper.emit[]` required and non-empty.
- Each emit entry requires `object_type`, `permission`, and one extractor:
  - `fields` for direct extraction from root pointers.
  - `from_array` for array fan-out extraction.

## 5.3 Pointer semantics (normative)

- Root pointer: RFC6901 pointer starting with `/`, evaluated on full JSON row.
- Item pointer: starts with `./`, evaluated against current array element (only
  valid under `from_array.fields`).
- Any other pointer format is invalid config.

## 5.4 Fallback semantics (normative)

- `fallback_paths` is a map from placeholder name to ordered root pointers.
- Fallback applies when a required placeholder is missing or empty.
- First non-empty fallback value wins.
- If no fallback produces a value, candidate remains missing and follows
  `missing_resource_key` behavior.

## 5.5 Example mapper files

Base mapper (`/data/metrics/.metricfs-map.yaml`):

```yaml
version: 1
rules:
  - match:
      glob: "*.jsonl"
    object_type: "metric_row"
    permission: "read"
    mapper:
      kind: "json_pointer"
      pointer: "/metric_row_id"
      canonical_template: "metric_row:{value}"
    missing_resource_key: "deny"
```

OpenLineage override (`/data/metrics/openlineage/.metricfs-map.yaml`):

```yaml
version: 1
extends: "../.metricfs-map.yaml"
rules:
  - match:
      glob: "*.jsonl"
    decision: "any"
    mapper:
      kind: "multi_extract"
      emit:
        - object_type: "dataset"
          permission: "read"
          from_array:
            pointer: "/event/inputs"
            fields:
              namespace: "./namespace"
              name: "./name"
            canonical_template: "dataset:{namespace}/{name}"
        - object_type: "dataset"
          permission: "read"
          from_array:
            pointer: "/event/outputs"
            fields:
              namespace: "./namespace"
              name: "./name"
            canonical_template: "dataset:{namespace}/{name}"
        - object_type: "job"
          permission: "read"
          fields:
            namespace: "/event/job/namespace"
            name: "/event/job/name"
          canonical_template: "job:{namespace}/{name}"
        - object_type: "run"
          permission: "read"
          fields:
            run_id: "/event/run/runId"
          canonical_template: "run:{run_id}"
      normalize:
        lowercase: true
        trim_slash: true
      fallback_paths:
        job_namespace:
          - "/event/facets/job/namespace"
        job_name:
          - "/event/facets/job/name"
        run_id:
          - "/event/facets/run/runId"
    missing_resource_key: "ignore"
```

## 6. SpiceDB model and transitive authorization

Transitive chain example (supported and expected):

- `user:alice -> orb:data_eng -> org:acme -> namespace:acme -> dataset/job/run`

This chain is explicitly represented in:

- `examples/spicedb-schema.zed`
- `examples/relationships.zed`

MVP expectation:

- Direct per-object grants are optional.
- Transitive grants alone must be sufficient for visibility.

## 7. Runtime CLI contract (no runtime YAML)

`metricfs` runtime settings are provided through CLI flags only.

## 7.1 Commands

```bash
metricfs mount ...
metricfs validate-flags ...
metricfs warm-index --source-dir /data/metrics
metricfs stats --mount /mnt/metrics-alice
```

## 7.2 `mount` flags

| Flag | Required | Default | Notes |
|---|---|---|---|
| `--source-dir` | yes | none | Must exist and be readable. |
| `--mount-dir` | yes | none | Must exist; mountpoint path. |
| `--subject` | yes | none | Subject string, e.g. `user:alice`. |
| `--read-only` | no | `true` | MVP must reject writable mode. |
| `--allow-other` | no | `false` | Standard FUSE behavior. |
| `--spicedb-endpoint` | yes | none | gRPC endpoint. |
| `--spicedb-token` | no | none | Explicit token; overrides env. |
| `--spicedb-token-env` | no | `SPICEDB_TOKEN` | Env var name used when token flag not provided. |
| `--spicedb-consistency` | no | `minimize_latency` | SpiceDB consistency mode. |
| `--watch-enabled` | no | `true` | Subscribe to SpiceDB watch stream. |
| `--watch-reconnect-backoff` | no | `100ms..5s` | Watch reconnect range. |
| `--reconcile-interval` | no | `30s` | Periodic full reconciliation cadence. |
| `--on-spicedb-unavailable` | no | `fail_closed` | `fail_closed` or `serve_stale`. |
| `--stale-snapshot-ttl` | no | `0s` | Only used with `serve_stale`; `0s` disables stale serving. |
| `--index-dir` | no | `$XDG_CACHE_HOME/metricfs` | Sidecar index/cache root. |
| `--index-format-version` | no | `1` | Index compatibility version. |
| `--index-hash` | no | `xxh3_64` | Candidate hash algorithm. |
| `--index-workers` | no | `num_cpu` | Index build worker count. |
| `--mapper-file-name` | no | `.metricfs-map.yaml` | Directory mapper filename. |
| `--mapper-resolution` | no | `nearest_ancestor` | MVP supports this value only. |
| `--mapper-inherit-parent` | no | `true` | Enable `extends` behavior. |
| `--missing-mapper` | no | `deny` | MVP supports `deny` only; other values are invalid. |
| `--missing-resource-key` | no | `deny` | Global default when rule omits value. |

## 7.3 CLI validation and exit codes

- `validate-flags` returns:
  - `0` valid
  - `2` invalid flag values, missing required flags, or preflight path errors
- `mount` startup returns:
  - `2` for argument/config validation errors
  - `3` for dependency startup failures (for example SpiceDB unavailable when
    `--on-spicedb-unavailable=fail_closed`)

## 8. Security and failure behavior

- Deny-by-default for parse/extraction failures unless explicitly configured.
- Unauthorized rows must never be emitted.
- Path traversal is prevented via canonical root checks.
- Startup default is fail-closed.
- If configured with `serve_stale`, stale permissions are bounded by
  `--stale-snapshot-ttl`; expiry reverts to deny for new opens.

## 9. Performance targets (MVP)

- Mount startup to ready: < 5s for 1M indexed lines (warm cache).
- Policy-to-visibility lag for new opens: < 2s P95.
- Additional open latency from evaluation metadata:
  - small file (<10k lines): < 15ms P95
  - medium file (<1M lines): < 150ms P95 (warm index)

## 10. Benchmark plan and pass/fail criteria (MVP gate)

This section defines the minimum throughput benchmark required before MVP is
considered acceptable.

## 10.1 Benchmark environment

- Local NVMe/SSD storage (no network filesystem).
- Single `metricfs` mount process.
- One active reader process for baseline throughput checks.
- Warm cache run only for pass/fail (index built, mapper loaded, permission
  snapshot loaded).

## 10.2 Benchmark datasets

Run all three profiles:

1. `uniform_allow_100`:
- ~5 GB JSONL file, average line size 400-800 bytes.
- 100% of lines visible for subject.

2. `mixed_allow_30`:
- Same file shape.
- ~30% lines visible with clustered visibility (typical segments).

3. `sparse_allow_5`:
- Same file shape.
- ~5% lines visible with fragmented visibility (worst-case-ish segment mapping).

## 10.3 Measurement method

For each profile:

1. Mount with production-like flags.
2. Warm once (not measured):

```bash
cat /mnt/metrics-alice/bench.jsonl > /dev/null
```

3. Measure 10 timed runs:

```bash
/usr/bin/time -p sh -c 'cat /mnt/metrics-alice/bench.jsonl > /dev/null'
```

4. Compute throughput per run:
- `MBps = visible_bytes / real_seconds / 1_000_000`

5. Report `p50` and `p95` MB/s across the 10 runs.

## 10.4 Pass/fail criteria

MVP passes only if all conditions hold:

1. `mixed_allow_30`: `p50 >= 15 MB/s` and `p95 >= 10 MB/s`
2. `sparse_allow_5`: `p50 >= 10 MB/s` and `p95 >= 8 MB/s`
3. `uniform_allow_100`: `p50 >= 20 MB/s`
4. No unauthorized rows appear in any benchmark output validation run.
5. Policy propagation target from section 9 remains satisfied during benchmark
   suite (`<2s` P95 for new opens).

## 10.5 Failure handling

If any pass criterion fails:

- MVP is blocked for release.
- Capture profile, file size, CPU, storage type, candidate density, and
  top-level bottleneck (index eval, segment mapping, FUSE read path, etc.).
- Re-run after optimization with same dataset and method.

## 11. MVP decisions (locked)

- Multi-candidate rows are first-class.
- Decision mode is rule-level only (`any|all`), not per candidate.
- Zero emitted candidates always deny.
- Snapshot-at-open consistency is required.
- Read-only mount only.

## 12. Implementation plan

Phase 1 (MVP):

1. FUSE read-only mirror + JSONL filtering path.
2. Mapper resolver/compiler with strict schema validation.
3. Multi-candidate index format and evaluator.
4. SpiceDB snapshot service (`LookupResources`, `Watch`, reconcile).
5. Visible segment builder + offset mapping.
6. CLI preflight/validation and observability.

Phase 2:

1. Index prewarm daemon mode.
2. Multi-subject mounts in one process.
3. Persistent permission snapshot cache.

## 13. Testing strategy

Unit tests:

- Mapper parsing/validation (pointer formats, `extends`, rule selection).
- Candidate emission + fallback semantics.
- `any|all` evaluator behavior including zero-candidate deny.
- Offset mapping correctness for `read(offset,len)`.

Integration tests:

- Transitive-only visibility (`user -> orb -> org -> namespace -> metric_row`).
- OpenLineage multi-entity row authorization.
- Policy update propagation on new opens.
- SpiceDB unavailable behavior (`fail_closed` vs `serve_stale`).

Property tests:

- Filtered output is a subsequence of original line set.
- No unauthorized candidate can make a line visible.
