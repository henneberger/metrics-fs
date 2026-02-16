# metricfs

`metricfs` is a mountable filesystem proxy for JSONL metrics that enforces
SpiceDB-style permissions at read time.

When a user reads files from the mounted path with normal CLI tools (`cat`,
`grep`, `awk`, `jq`, etc.), they only see rows they are allowed to read.
Unauthorized rows are omitted, not redacted.

## Why this exists

- Keep the AI/tooling workflow simple: use files + shell tools.
- Preserve centralized authorization logic in SpiceDB.
- Apply policy changes quickly without changing source data.

## Core idea

1. Source JSONL files remain unchanged on disk.
2. A FUSE mount presents a filtered virtual view.
3. A local mapper layer canonicalizes heterogeneous metric rows into stable
   resource keys.
4. A local permission snapshot (`allowed resource IDs`) is kept current from
   SpiceDB (watch + periodic reconciliation).
5. Reads stream only authorized lines.

## Mapping model

- Mapping and normalization live in `metricfs` (fast local transforms).
- Authorization graph and inheritance live in SpiceDB.
- Example: OpenLineage rows can emit multiple canonical keys (input/output
  datasets, job, run), and `metricfs` applies a decision mode (for example
  `any`) across those permission checks.
- Transitive chains are supported by the auth graph, for example
  `user -> orb -> org -> namespace -> dataset/job/run`.
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
No production binary is included yet.
