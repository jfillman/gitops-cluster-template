# 40-observability

> **Template note**: this file was carried over from `gitops-cluster-dev`'s own README and describes that cluster's real build history — useful background on how this component works and why, not a to-do list for your new cluster. Treat dates/"live-verified" claims as historical, not something to re-verify on this one.


Prometheus/Grafana/Tempo/Loki stack — per `gitops-strategy.md` §3.

## Real, built, live-verified on `kind-dev` (2026-08-13)

Full stack, mirroring `kind-observe`'s own (per direct instruction, superseding an
earlier, deliberately-minimal first pass of `kube-prometheus-stack/application.yaml`
that skipped Thanos/Loki/Tempo/MinIO/OTel Collector entirely) - `minio/`,
`kube-prometheus-stack/`, `thanos/`, `loki/`, `tempo/`, `otel-collector/`, each a
real Application, values transcribed from `/Users/jerf/tech/observability/*-values.yaml`
(the actual source design `kind-observe`'s stack was built from - see that
directory's own `README.md` for the full architecture diagram and rationale).

**Sync order matters** (`SYNCPOLICY: Manual` throughout, same as `10-crds-operators/` -
no automated dependency ordering, sync by hand in this order): `minio` (bucket +
`thanos-objstore-config` Secret) → `kube-prometheus-stack` (Thanos sidecar needs the
Secret) → `thanos` (Query needs `kube-prometheus-stack-thanos-discovery`) → `loki` +
`tempo` (need their MinIO buckets) → `otel-collector` (needs Tempo/Loki's endpoints
up to have anywhere to export to, though it'll come up regardless - only the
pipeline's actual delivery depends on this order, not the collector's own health).

**One real chart-provenance catch, worth recording**: the source README documented
*two* alternative ways to install MinIO (`bitnami/minio` or the community
`minio/minio` from `charts.min.io`) without saying which was actually used.
`kind-observe`'s real live version (`5.4.0`, this file's own table below) only
resolves against the community chart's version numbering - `bitnami/minio`'s
current version range (15.x) doesn't overlap with `5.4.0` at all, confirmed via
`helm search repo` before picking - so `minio/application.yaml` uses
`charts.min.io`, not bitnami.

Release name is exactly `kube-prometheus-stack` deliberately - the chart's own
default `release:`-label ServiceMonitor/PrometheusRule selectors line up with what
idp-service-catalog's SLO Composition and `idp-application`'s own
`serviceMonitor.additionalLabels` already assumed - confirmed against a real
install now, not left as an unconfirmed placeholder. `ServerSideApply=true` on
`kube-prometheus-stack` (same annotation-size-limit fix `01-argocd-platform/`/
`external-secrets/` needed - its CRDs are large enough to hit it too).

**HolmesGPT is not built here yet** - its real values (narrower toolset, dropped
Grafana/Loki+Tempo, `github` MCP addon - see `idp_session_phase2_holmesgpt`) were
never captured into a file anywhere findable on disk, so reconstructing it accurately
needs real work, not a guess. Deferred, not forgotten.

**Two more real bugs, both in `minio/minio` (community, `charts.min.io`) 5.4.0
specifically, both fixed live**: (1) the source README's `--set
defaultBuckets="thanos,loki,tempo"` doesn't match this chart version's actual
values schema at all - silently ignored (Helm doesn't error on unknown keys), real
field is `buckets:` (a list of `{name, policy}` objects); (2) even with `buckets:`
set correctly, the chart's own post-install Job never actually creates them - its
init script defines a `createBucket()` shell function but never calls it, confirmed
via a clean `helm template` render, not a deployment fluke. Worked around with a
plain `create-buckets-job.yaml` (`mc mb --ignore-existing`) rather than patching
the chart. Downstream effect while these were being tracked down:
`tempo-0`/`thanos-storegateway-0` show real restart counts in their pod history
(crash-looped on "bucket does not exist" before the buckets existed) - both
stable/healthy once the buckets landed; the restart counts are expected history,
not a live problem.

## kind-observe's own state (documented, not yet re-templated or adopted)

This section describes `kind-observe` specifically, the cluster this repo's own
top-level README names as its primary target - **not** `kind-dev` above. Six Helm
releases already live in the `observability` namespace there, several with
substantial custom values (`kube-prometheus-stack` wires real Grafana datasource
integration to Loki/Tempo/Thanos) - capturing all of them with the same fidelity as
`10-crds-operators` (exact extracted values, live-verified non-destructive adoption) is
real work, deliberately scoped out of this pass rather than rushed. **Also true as of
this same pass: `kind-observe`'s own ArgoCD has no Applications in it yet either
(confirmed live) - the "adopted" pieces documented in `10-crds-operators/` were
values-matched, not actually synced there. `kind-dev`, bootstrapped fresh this pass,
is the first cluster this repo's Applications have actually been live-synced
against.**

| Release | Chart | Version |
|---|---|---|
| `kube-prometheus-stack` | `kube-prometheus-stack` | 87.19.1 (app v0.92.1) |
| `loki` | `loki` | 7.1.0 (app 3.6.8) |
| `minio` | `minio` | 5.4.0 |
| `otel-collector` | `opentelemetry-collector` | 0.165.0 |
| `tempo` | `tempo` | 1.24.4 (app 2.9.0) |
| `thanos` | `thanos` | 17.3.1 (app 0.39.2) |

**`holmesgpt`** (chart `holmes` 0.38.0) also runs in its own `holmesgpt` namespace —
found during this cluster inventory, not previously known to any `idp` design doc.
Robusta's AI-powered K8s troubleshooting/root-cause tool — directly adjacent to the
`ai-rollout` diagnosis-job mechanism (goal 9, "AI embedded at the control-plane layer")
that's already been folded into the `idp` plan. **Not yet understood well enough to say
whether it's meant to integrate with that work or is a separate exploration** — worth
asking about directly rather than assuming either way before Phase 2's AI-triage design
work happens.
