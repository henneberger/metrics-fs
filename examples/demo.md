# Example usage

This example demonstrates expected behavior for `user:alice`.

## 1. Prepare source metrics

```bash
mkdir -p /data/metrics
cp /Users/henneberger/libs/metricfs/examples/metrics/orders.jsonl /data/metrics/orders.jsonl
cp /Users/henneberger/libs/metricfs/examples/metrics/.metricfs-map.yaml /data/metrics/.metricfs-map.yaml
mkdir -p /data/metrics/openlineage
cp /Users/henneberger/libs/metricfs/examples/metrics/openlineage/.metricfs-map.yaml /data/metrics/openlineage/.metricfs-map.yaml
mkdir -p /mnt/metrics-alice
```

## 2. Start SpiceDB and load schema

```bash
# Example only. Adjust for your environment.
spicedb serve --grpc-preshared-key "$SPICEDB_TOKEN"
zed schema write /Users/henneberger/libs/metricfs/examples/spicedb-schema.zed
zed relationship import /Users/henneberger/libs/metricfs/examples/relationships.zed
```

The sample relationships include namespace inheritance and a transitive chain
through orb and org membership.

Transitive example included in the sample:

- `user:alice -> orb:data_eng -> org:acme -> namespace:acme -> metric_row:orders_1`
- Alice can read `orders_1` and `orders_3` via this transitive path (without
  direct per-row grants).

## 3. Mount filtered view for alice

```bash
metricfs mount \
  --source-dir /data/metrics \
  --mount-dir /mnt/metrics-alice \
  --subject user:alice \
  --read-only \
  --allow-other=false \
  --spicedb-endpoint 127.0.0.1:50051 \
  --spicedb-token-env SPICEDB_TOKEN \
  --mapper-file-name .metricfs-map.yaml \
  --mapper-resolution nearest_ancestor \
  --mapper-inherit-parent \
  --missing-mapper deny \
  --missing-resource-key deny
```

`metricfs` auto-discovers mapping rules from per-directory `.metricfs-map.yaml`
files. No global dataset list is required.

## 4. Query with normal shell tools

```bash
cat /mnt/metrics-alice/orders.jsonl
```

Expected output:

```json
{"metric_row_id":"orders_1","tenant":"acme","metric":"latency_ms","value":112}
{"metric_row_id":"orders_3","tenant":"acme","metric":"error_rate","value":0.021}
```

Regular tooling works as-is:

```bash
awk -F'"' '/tenant/ {print $8}' /mnt/metrics-alice/orders.jsonl
grep '"metric":"error_rate"' /mnt/metrics-alice/orders.jsonl
jq -c '. | select(.value > 0.02)' /mnt/metrics-alice/orders.jsonl
```

## 5. Change policy and see update quickly

Grant alice access to one more row:

```bash
zed relationship create metric_row:orders_4#viewer@user:alice
```

New opens should reflect the policy update within seconds:

```bash
cat /mnt/metrics-alice/orders.jsonl
```

Expected output after update:

```json
{"metric_row_id":"orders_1","tenant":"acme","metric":"latency_ms","value":112}
{"metric_row_id":"orders_3","tenant":"acme","metric":"error_rate","value":0.021}
{"metric_row_id":"orders_4","tenant":"delta","metric":"latency_ms","value":143}
```

## 6. OpenLineage mapping model

For OpenLineage-style rows, place a directory mapper file at
`/data/metrics/openlineage/.metricfs-map.yaml` that emits multiple canonical
keys per event:

- All input datasets from `/event/inputs[*]`
- All output datasets from `/event/outputs[*]`
- Job object from `/event/job/{namespace,name}`
- Run object from `/event/run/runId`

The sample mapper uses `decision: any`, so the line is visible if the subject
has `read` on at least one emitted object (dataset/job/run). It also includes
fallback paths for job and run fields in facets when top-level fields are
missing.

Use namespace inheritance in SpiceDB to avoid per-dataset grants:

```zed
definition dataset {
  relation viewer: user
  relation parent_namespace: namespace
  permission read = viewer + parent_namespace->read
}
```
