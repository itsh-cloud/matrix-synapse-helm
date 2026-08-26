# ITSH fork of the matrix-synapse chart

This is the `matrix-synapse` chart from
[gitlab.com/ananace/charts](https://gitlab.com/ananace/charts), extracted into a standalone
repository and carrying two changes. Everything else is upstream.

Base: **3.12.36** (appVersion 1.159.0). Our versions are tagged `3.12.36-itsh.N`.

Upstream is Apache 2.0, Copyright 2021 Alexander Olofsson. `LICENSE` is retained unchanged.
Per Apache 2.0 §4(b), the modified files carry `ITSH FORK CHANGE` notices at each edit and
the changes are described below.

## Change 1: Redis is conditional on workers

`templates/secrets.yaml`

Synapse only needs Redis for **worker replication**. A monolith with no workers does not use
it. Upstream nonetheless rendered

```yaml
redis:
  enabled: true
  host: ...
```

into `homeserver.yaml` unconditionally, and `matrix-synapse.redis.host` hard-fails with
`A valid externalRedis.host is required` when `redis.enabled` is false. The combination made
a Redis mandatory for every deployment, including 0-worker monoliths that never touch it.

The block is now gated on whether any worker is enabled, using the same idiom the chart
already uses in `worker-configuration.yaml`:

```gotemplate
{{- $anyWorker := false }}
{{- range $worker, $config := .Values.workers }}
  {{- if $config.enabled }}{{- $anyWorker = true }}{{- end }}
{{- end }}
{{- if $anyWorker }}
```

With no enabled workers the chart renders no redis config, the helper is never evaluated, and
no Redis is needed. Enable a worker and upstream behaviour returns unchanged, including the
`externalRedis.host` requirement.

Verified by rendering both ways: monolith gives 14 resources and zero redis blocks; with
`workers.appservice.enabled=true` it gives 16 resources and one redis block.

## Change 2: no bitnami

`Chart.yaml`, `values.yaml`

The `postgresql` and `redis` subchart dependencies pointed at `charts.bitnami.com`, and the
default images at `bitnamilegacy/*`. Bitnami's free distribution is being wound down and the
images have already moved to the `bitnamilegacy` archive, so depending on either is a
liability rather than a convenience.

Both dependencies are removed. `values.yaml` keeps the now-inert `postgresql.image` and
`redis.image` keys as empty strings with a note, rather than deleting them, so a values file
written against upstream does not fail to parse. `signingKey.publishImage` moved from
`bitnami/kubectl` to `registry.k8s.io/kubectl`.

Use `externalPostgresql`, and `externalRedis` if you run workers.

This is also what lets the chart render with no `helm dependency build` at all: nothing is
fetched from a third-party chart repo at render time.

## Re-syncing with upstream

Upstream is a monorepo. The chart-only history was produced with:

```bash
git clone https://gitlab.com/ananace/charts.git
cd charts
git subtree split --prefix=charts/matrix-synapse -b matrix-synapse-only
```

To pull a newer upstream chart, re-run that split, add it as a remote here, and rebase the
`ITSH FORK CHANGE` commits onto it. Both changes are small and localised, so conflicts should
be limited to `templates/secrets.yaml`, `Chart.yaml` and `values.yaml`.

Bump `version` to the new upstream version with an `-itsh.1` suffix when you do.
