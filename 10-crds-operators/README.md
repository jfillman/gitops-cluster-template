# 10-crds-operators

> **Template note**: this file was carried over from `gitops-cluster-dev`'s own README and describes that cluster's real build history — useful background on how this component works and why, not a to-do list for your new cluster. Treat dates/"live-verified" claims as historical, not something to re-verify on this one.


Crossplane + Providers/Configurations, cert-manager, External Secrets Operator,
ingress — per `gitops-strategy.md` §3.

## Adopted, real, live-verified this pass

- `crossplane/` — the Crossplane controller itself (Helm), `provider-kubernetes`, and
  the four Functions in use (`function-auto-ready`, `function-go-templating`,
  `function-patch-and-transform`, `function-rollout-watcher`) - every Composition in
  this catalog, including the SLO one, shares this single `function-go-templating`
  registration via `source: Inline` (templates embedded directly in each
  Composition). **Do not register a second `Function` object pointing at a package
  reference already installed here** - tried that for the SLO Composition
  (isolating its templates via a dedicated mount) and it corrupted Crossplane
  v2.3.4's package-manager dependency-lock graph for every other Function on the
  cluster, found live when `function-auto-ready` lost its runtime Deployment
  entirely. `crossplane/native-resources-rbac.yaml` also lives here - Crossplane's
  controller `ServiceAccount` needs an explicit grant per native resource kind any
  Composition composes directly (currently `argoproj.io` Rollout/AnalysisTemplate,
  `batch` Job, `monitoring.coreos.com` PrometheusRule, `sloth.slok.dev`
  PrometheusServiceLevel) - it has no built-in RBAC for these, only its own API
  types.
- `sloth/` — Sloth (sloth.dev), installed straight from its own git repo path (not
  published to a Helm chart repo). Watches `PrometheusServiceLevel` CRs and
  generates the multi-window-multi-burn-rate `PrometheusRule` for each - the SLO
  Composition generates the CR, not the rule directly. See the Application's own
  header comments for the `sloth.extraLabels` gotcha (stamps labels onto individual
  generated rules, not the `PrometheusRule` object itself - that's on the
  Composition to set instead).
- `cert-manager/` — Helm, `crds.enabled: true` only, no other customization.
- `external-secrets/` — Helm, fully default values.

Each renders identically to what was already running (same chart, same pinned version,
same extracted values) — adoption should be a no-op sync, not a recreation. Verify via
unchanged resource creation timestamps after the app-of-apps root goes live, same check
used in `platform_cicd_session_argocd_onboarding`.

## Vendored + built (2026-08-13, live-verified on `kind-dev`)

- **`argo-rollouts/`** — vendored `install.yaml` pinned to v1.9.1 (was previously
  installed from a `.../releases/latest/download/...` URL - not reproducible, not
  really GitOps-safe). `ServerSideApply=true` from the start, given the pattern's
  now been hit three separate times on this cluster's other CRD-heavy installs.
- **`contour/`** — vendored `install.yaml` pinned to v1.32.1, namespace
  `projectcontour`. This is the cluster's ingress controller - first time it's been
  named explicitly in any `idp` design doc (`gitops-strategy.md` referred to
  "ingress" generically without knowing which one this cluster already runs); also
  the real value `idp-application`'s `networkPolicy.ingressControllerNamespaceSelector`
  had only as an explicitly-flagged, unconfirmed `ingress-nginx` guess - fixed there
  once this was confirmed live.
- **`crossplane/providers.yaml`'s new `provider-github` entry + `provider-github-config.yaml`**
  (2026-08-13) — backs `NodeJSApplication` (`idp-service-catalog`, the first Bootstrap-tier XRD,
  `idp/docs/service-catalog-design.md` §1). Real package name confirmed live against
  the provider's own source, not guessed: `crossplane-contrib/provider-upjet-github`.
  Two real corrections found live-verifying `NodeJSApplication`, both detailed in
  `provider-github-config.yaml`'s own header and `service-catalog-design.md` §1: the
  namespaced managed-resource family (`repo.github.m.upbound.io`), not the Cluster-scoped
  one (Crossplane v2 rejects a namespaced XR composing a Cluster-scoped resource
  outright), and a PAT, not `platform-cicd`'s existing GitHub App (GitHub Apps can't
  create repos under `jfillman`'s personal account at all - the Secret itself is never
  committed here).

## Built + live-verified end-to-end, 2026-08-17

- **`infisical/`** — Infisical Community Edition, self-hosted, `idp/docs/
  service-catalog-design.md` Item 8's platform-infrastructure half. Chart identity
  (`infisical-standalone` 1.10.0, Cloudsmith-hosted) confirmed against the real
  published index, not the misleading local folder name upstream uses. Real findings
  from tracing the chart's actual templates, all in the Application's own header:
  `ingress.enabled: false` alone doesn't skip the bundled ingress-nginx controller;
  bundled Postgres/Redis passwords can't be routed through `existingSecret` without
  breaking the app container's own connection-string env var; `autoBootstrap`'s
  default `infisical/cli` image tag (`0.41.86`, no arch suffix) crashes on every
  invocation on this arm64 host with a Go runtime fatal error - its "arm64" manifest
  entry actually contains amd64 binary content, runs under transparent QEMU
  emulation, hits a known Go-runtime-under-emulation crash - fixed by pinning the
  explicit `-arm64`-suffixed tag instead. Requires a manually-created
  `infisical-secrets` Secret (`AUTH_SECRET`/`ENCRYPTION_KEY`/`SITE_URL`) and an
  `infisical-bootstrap-credentials` Secret (one-time admin login, consumed once by
  the bootstrap Job) - see the Application's header for both commands.
- **`infisical-secretstore-operator/`** — kopf (Python) controller reconciling
  `InfisicalProject` CRs against Infisical's real REST API (project + machine
  identity + Universal Auth per (app,cluster) pair) - source in
  `idp-service-catalog/operators/`. No registry yet - `kind load image-archive`'d
  directly. Real bugs found live, not caught by any offline check: kopf's
  `getpass.getuser()` crashes for a bare numeric UID with no `/etc/passwd` entry
  (fixed with a `USER` env var + `--standalone`); a JWT-claim-decode approach to
  discovering the Infisical org id doesn't work against real tokens (no such claim
  exists) - replaced with a one-time-looked-up `INFISICAL_ORG_ID` constant;
  `imagePullPolicy: Never` needs the image ref's `localhost/` prefix explicitly.
- **`crossplane/provider-kubernetes-config.yaml`** (new) — `provider-kubernetes` was
  installed but never configured until the `SecretStore` Composition needed it:
  Crossplane v2 rejects composing a cluster-scoped native resource
  (`ClusterSecretStore`) directly from a namespaced XR, same restriction
  `NodeJSApplication`'s `provider-github` already hit for Cluster-scoped repo
  resources - same fix, the namespaced `Object` family
  (`kubernetes.m.crossplane.io`), wrapping the manifest in
  `spec.forProvider.manifest` instead of composing it directly.

Full chain live-proven on `kind-dev`, not just "resources exist": a real
`SecretStore` XR → a real Infisical project + environment + machine identity +
Universal Auth credentials (via `infisical-secretstore-operator`) → a real,
`Ready: True` `ClusterSecretStore` → a real secret written to Infisical's own API →
pulled by a real `ExternalSecret` into a real Kubernetes Secret with the correct
value. Also surfaced a real, pre-existing bug in the already-shipped
`idp-application` chart's `ExternalSecret` template (`remoteRef.property` set to
the secret's own name breaks every pull against Infisical's flat key-value secrets -
fixed, `idp-service-catalog` v0.3.16).

**Not yet built**: wiring `SecretStore` into `ApplicationEnvironment`'s
auto-provisioning (create-on-first-env-for-a-cluster, reference on later ones) -
this XRD is standalone-creatable only, same as `SLO` before any Attached-tier
auto-provisioning existed. Real design discussion happened for this (idempotent
git-file-write for the "N sibling XRs, one shared resource" problem, `Usage` for
deletion-safety, both reusing patterns already live in this catalog) but wasn't
implemented - separate follow-up.
