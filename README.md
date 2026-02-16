# metricfs

`metricfs` is a mountable filesystem proxy for JSONL metrics that enforces
SpiceDB-style permissions at read time.

When a user reads files from the mounted path with normal CLI tools (`cat`,
`grep`, `awk`, `jq`, etc.), they only see rows they are allowed to read.
Unauthorized rows are omitted, not redacted.

## Simple example

Input JSONL:

```json
{"metric_row_id":"1","value":10}
{"metric_row_id":"2","value":20}
{"metric_row_id":"3","value":30}
```

If the subject can read rows `1` and `3`, then:

```bash
cat /mnt/metrics/orders.jsonl
```

returns:

```json
{"metric_row_id":"1","value":10}
{"metric_row_id":"3","value":30}
```

## Why this exists

- Keep the AI/tooling workflow simple: use files + shell tools.
- Preserve centralized authorization logic in SpiceDB.
- Apply policy changes quickly without changing source data.

## Core idea

1. Source JSONL files remain unchanged on disk.
2. A FUSE mount presents a filtered virtual view.
3. A local mapper layer canonicalizes heterogeneous metric rows into stable
   resource keys.
4. Authorization checks come from a configurable backend:
   - `file`: local explicit allow-list JSON.
   - `spicedb`: live checks against SpiceDB.
5. Reads stream only authorized lines.

## Auth backends

`metricfs` supports two auth modes:

- `file`
  - Fast local development.
  - Uses `--permissions-file` JSON allow-list.
  - No `--subject` required by this backend.
- `spicedb`
  - Uses live checks against SpiceDB.
  - Requires `--subject`, `--spicedb-endpoint`, and token
    (`--spicedb-token` or env via `--spicedb-token-env`).

## Mapping model

- Mapping and normalization live in `metricfs` (fast local transforms).
- Authorization graph and inheritance live in SpiceDB.
- Example: OpenLineage rows can emit a canonical job key
  (`job:{namespace}/{name}`), and `metricfs` applies a decision mode (for
  example `any`) across emitted checks.
- Transitive chains are supported by the auth graph, for example
  `user -> orb -> org -> namespace -> job`.
- Mapper rules are discovered per directory from `.metricfs-map.yaml`, so teams
  can add new metric formats without editing one global dataset list.

## MVP capabilities

- Per-subject mount (for example one mount per human/user/service account).
- Row-level filtering for `*.jsonl`.
- Fast policy update propagation (target: visible within seconds for new reads).
- Sidecar file index for line offsets + resource IDs to avoid reparsing full JSON
  on every read.
- Concrete throughput benchmark gate in spec (`p50 >= 15 MB/s`, `p95 >= 10 MB/s`
  for mixed-visibility profile).

## Project layout

- `docs/spec.md`: detailed architecture and behavior spec.
- `examples/metrics/.metricfs-map.yaml`: directory mapper config.
- `examples/metrics/openlineage/.metricfs-map.yaml`: OpenLineage override.
- `examples/spicedb-schema.zed`: sample SpiceDB schema.
- `examples/relationships.zed`: sample relationships.
- `examples/metrics/orders.jsonl`: sample metrics file.
- `examples/demo.md`: end-to-end usage walkthrough.

## Design status

This project currently contains an MVP-ready specification and usage model.
An initial implementation is now included (Go CLI).

Current implementation mode:

- `metricfs mount` uses real FUSE when available.
- `metricfs render --file ...` provides a non-FUSE filtered read path for
  environments where FUSE is unavailable.

## Docker + FUSE

You can run `metricfs mount` in a Linux container with FUSE enabled.

Build:

```bash
docker build -t metricfs:fuse .
```

Run (interactive mount session):

```bash
docker run --rm -it \
  --privileged \
  --device /dev/fuse:/dev/fuse \
  --cap-add SYS_ADMIN \
  --security-opt apparmor:unconfined \
  -v "$(pwd)/examples:/workspace/examples:ro" \
  -v /tmp/metricfs-mnt:/mnt/metricfs \
  metricfs:fuse \
  metricfs mount \
    --source-dir /workspace/examples/metrics \
    --mount-dir /mnt/metricfs \
    --auth-backend file \
    --permissions-file /workspace/examples/permissions-alice.json \
    --read-only \
    --mapper-file-name .metricfs-map.yaml \
    --mapper-resolution nearest_ancestor \
    --mapper-inherit-parent \
    --missing-mapper deny \
    --missing-resource-key deny
```

Quick self-test (runs mount + `cat`/`grep` inside the container):

```bash
docker run --rm \
  --privileged \
  --device /dev/fuse:/dev/fuse \
  --cap-add SYS_ADMIN \
  --security-opt apparmor:unconfined \
  -v "$(pwd)/examples:/workspace/examples:ro" \
  metricfs:fuse \
  sh -lc 'mkdir -p /mnt/metricfs && metricfs mount \
    --source-dir /workspace/examples/metrics \
    --mount-dir /mnt/metricfs \
    --auth-backend file \
    --permissions-file /workspace/examples/permissions-alice.json \
    --read-only \
    --mapper-file-name .metricfs-map.yaml \
    --mapper-resolution nearest_ancestor \
    --mapper-inherit-parent \
    --missing-mapper deny \
    --missing-resource-key deny >/tmp/mount.log 2>&1 & pid=$!; \
    sleep 2; \
    echo "== CAT =="; cat /mnt/metricfs/orders.jsonl; \
    echo "== GREP error_rate =="; grep "\"metric\":\"error_rate\"" /mnt/metricfs/orders.jsonl; \
    echo "== GREP orders_2 (should be empty) =="; grep "orders_2" /mnt/metricfs/orders.jsonl || true; \
    kill $pid >/dev/null 2>&1 || true; wait $pid >/dev/null 2>&1 || true'
```

If you want mounted output on the host filesystem, bind-mount a host directory
to `/mnt/metricfs` in the container:

```bash
docker run --rm -it \
  --privileged \
  --device /dev/fuse:/dev/fuse \
  --cap-add SYS_ADMIN \
  --security-opt apparmor:unconfined \
  -v "$(pwd)/examples:/workspace/examples:ro" \
  -v /tmp/metricfs-mnt:/mnt/metricfs \
  metricfs:fuse \
  metricfs mount \
    --source-dir /workspace/examples/metrics \
    --mount-dir /mnt/metricfs \
    --auth-backend file \
    --permissions-file /workspace/examples/permissions-alice.json \
    --read-only \
    --mapper-file-name .metricfs-map.yaml \
    --mapper-resolution nearest_ancestor \
    --mapper-inherit-parent \
    --missing-mapper deny \
    --missing-resource-key deny
```

Then from the host:

```bash
cat /tmp/metricfs-mnt/orders.jsonl
```

### File backend (Docker/FUSE)

Use `--auth-backend file` with permissions JSON:

```bash
docker run --rm -it \
  --privileged \
  --device /dev/fuse:/dev/fuse \
  --cap-add SYS_ADMIN \
  --security-opt apparmor:unconfined \
  -v "$(pwd)/examples:/workspace/examples:ro" \
  -v /tmp/metricfs-mnt:/mnt/metricfs \
  metricfs:fuse \
  metricfs mount \
    --source-dir /workspace/examples/metrics \
    --mount-dir /mnt/metricfs \
    --auth-backend file \
    --permissions-file /workspace/examples/permissions-alice.json \
    --read-only \
    --mapper-file-name .metricfs-map.yaml \
    --mapper-resolution nearest_ancestor \
    --mapper-inherit-parent \
    --missing-mapper deny \
    --missing-resource-key deny
```

### SpiceDB backend (Docker/FUSE)

Use `--auth-backend spicedb` and live SpiceDB checks:

```bash
docker run --rm -it \
  --privileged \
  --device /dev/fuse:/dev/fuse \
  --cap-add SYS_ADMIN \
  --security-opt apparmor:unconfined \
  -v "$(pwd)/examples:/workspace/examples:ro" \
  -v /tmp/metricfs-mnt:/mnt/metricfs \
  metricfs:fuse \
  metricfs mount \
    --source-dir /workspace/examples/metrics \
    --mount-dir /mnt/metricfs \
    --auth-backend spicedb \
    --spicedb-endpoint http://host.docker.internal:8443 \
    --spicedb-token testtoken \
    --subject user:alice \
    --read-only \
    --mapper-file-name .metricfs-map.yaml \
    --mapper-resolution nearest_ancestor \
    --mapper-inherit-parent \
    --missing-mapper deny \
    --missing-resource-key deny
```

## OpenTelemetry example (non-FUSE)

This repo includes a generic OpenTelemetry JSONL example:

- data: `examples/metrics/opentelemetry/otel-metrics.jsonl`
- mapper: `examples/metrics/opentelemetry/.metricfs-map.yaml`
- permissions: `examples/opentelemetry-permissions-alice.json`

Run:

```bash
./bin/metricfs render \
  --file examples/metrics/opentelemetry/otel-metrics.jsonl \
  --source-dir examples/metrics \
  --auth-backend file \
  --permissions-file examples/opentelemetry-permissions-alice.json \
  --mapper-file-name .metricfs-map.yaml \
  --mapper-resolution nearest_ancestor \
  --mapper-inherit-parent \
  --missing-mapper deny \
  --missing-resource-key deny
```

This path demonstrates generalized filtering with arbitrary object types
(`telemetry_item`) using mapper + permissions input, without relying on
object-type-specific tuple evaluation logic.

## SpiceDB backend quickstart (non-FUSE)

Start SpiceDB (testing mode) in Docker:

```bash
docker rm -f metricfs-spicedb-test >/dev/null 2>&1 || true
docker run -d --name metricfs-spicedb-test \
  -p 50051:50051 -p 8443:8443 \
  authzed/spicedb serve-testing --http-enabled
```

Load schema:

```bash
docker run --rm -v "$(pwd)/examples:/examples:ro" authzed/zed:latest \
  schema write /examples/spicedb-schema.zed \
  --endpoint host.docker.internal:50051 \
  --token testtoken \
  --insecure
```

Load relationships:

```bash
while IFS= read -r line; do
  line="${line%$'\r'}"
  [[ -z "$line" || "${line:0:1}" == "#" ]] && continue
  resource_part="${line%%@*}"; subject_part="${line#*@}"
  resource="${resource_part%%#*}"; relation="${resource_part#*#}"
  res_type="${resource%%:*}"; res_id="${resource#*:}"
  subject_obj="${subject_part%%#*}"; subj_rel=""
  [[ "$subject_part" == *"#"* ]] && subj_rel="${subject_part#*#}"
  sub_type="${subject_obj%%:*}"; sub_id="${subject_obj#*:}"
  payload=$(jq -nc --arg rt "$res_type" --arg rid "$res_id" --arg rel "$relation" --arg st "$sub_type" --arg sid "$sub_id" --arg srel "$subj_rel" \
    '{updates:[{operation:"OPERATION_TOUCH",relationship:{resource:{objectType:$rt,objectId:$rid},relation:$rel,subject:{object:{objectType:$st,objectId:$sid}}}}]} | if $srel != "" then .updates[0].relationship.subject.optionalRelation = $srel else . end')
  curl -sS -f -X POST 'http://127.0.0.1:8443/v1/relationships/write' \
    -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer testtoken' \
    --data "$payload" >/dev/null
done < examples/relationships.zed
```

Run `metricfs` against live SpiceDB:

```bash
./bin/metricfs render \
  --auth-backend spicedb \
  --spicedb-endpoint http://127.0.0.1:8443 \
  --spicedb-token testtoken \
  --subject user:alice \
  --source-dir examples/metrics \
  --file examples/metrics/openlineage/events.jsonl \
  --mapper-file-name .metricfs-map.yaml \
  --mapper-resolution nearest_ancestor \
  --mapper-inherit-parent \
  --missing-mapper deny \
  --missing-resource-key deny
```

Expected visible OpenLineage rows for `user:alice`: `ol_1_job_allowed`,
`ol_3_job_allowed_via_normalize`, and `ol_4_facets_job_only`
(transitive inheritance and job facets fallback), while `ol_2_denied` remains
hidden.

## Throughput benchmark (SpiceDB)

Measured with live SpiceDB checks and a transitive chain:

- Chain: `user:alice -> orb:data_eng -> org:acme -> namespace:acme -> job:prod/airflow/daily_etl`
- Workload: `300,000` OpenLineage rows (alternating allow/deny jobs)
- Input size: `39,488,895` bytes (~`37.7 MiB`)
- Visible rows: `150,000`

`metricfs render --auth-backend spicedb` results:

- Run 1: `27.66 MB/s`
- Run 2: `59.43 MB/s`
- Run 3: `59.58 MB/s`
- Run 4: `54.28 MB/s`
- Run 5: `59.57 MB/s`

This exceeds the MVP gate in the spec (`p50 >= 15 MB/s`, `p95 >= 10 MB/s`).

Compose option (SpiceDB-backed mount):

```bash
docker compose -f docker-compose.fuse.yml up --build
```

This compose file starts:

1. `spicedb` (testing mode)
2. `spicedb-seed` (loads schema + relationships from `examples/`)
3. `metricfs` mount with `--auth-backend spicedb`

Important: Docker Desktop on macOS typically does not expose `/dev/fuse` from
the host kernel in a way that supports this pattern. If that applies, run this
setup on a Linux VM/host and access the mounted output there (or use
`metricfs render` on macOS).
