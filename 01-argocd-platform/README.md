# 01-argocd-platform

> **Template note**: this file was carried over from `gitops-cluster-dev`'s own README and describes that cluster's real build history — useful background on how this component works and why, not a to-do list for your new cluster. Treat dates/"live-verified" claims as historical, not something to re-verify on this one.


ArgoCD's own install + self-management — per `gitops-strategy.md` §4, "let ArgoCD
manage ArgoCD." Renamed from `01-argocd` 2026-08-16 when `argocd-apps` was split out
(see status below) - pure repo reorg, zero live effect (`install.yaml` was never
discovered by `root-app-of-apps.yaml`'s own recurse glob, so nothing was reading this
directory's old name to begin with).

## Current state (documented, not yet self-managed)

`install.yaml` in this directory is the vendored, pinned upstream manifest
(`https://raw.githubusercontent.com/argoproj/argo-cd/v3.4.5/manifests/install.yaml`,
`quay.io/argoproj/argocd:v3.4.5`) — not an ArgoCD `Application` object, deliberately
(see "why not an Application" below). This is now a real, reproducible artifact,
confirmed live on `kind-dev` (2026-08-13): a from-scratch bootstrap, not just an
adoption of already-running state, the first time this exact procedure has been
proven end-to-end.

**Bootstrap steps, in order** (the one manual sequence before everything else becomes
a normal PR — same framing as `root-app-of-apps.yaml`'s own header comment):

```
kubectl create namespace argocd
kubectl apply --server-side -n argocd -f 01-argocd-platform/install.yaml
kubectl apply -f root-app-of-apps.yaml
```

**`--server-side`, not plain `apply` - a real bug hit live, not a style choice.**
Plain `kubectl apply` failed outright: `applicationsets.argoproj.io` (one of
ArgoCD's own CRDs) is too large for the `kubectl.kubernetes.io/last-applied-configuration`
annotation's 262144-byte limit (`metadata.annotations: Too long`). This is the exact
same failure mode later hit again, live, syncing `external-secrets`' CRDs through
ArgoCD itself - see that directory's `application.yaml` for the `ServerSideApply=true`
syncOption that's the equivalent fix for an ArgoCD-managed Application (this vendored
manifest isn't ArgoCD-managed, so the fix here is the `kubectl` flag directly, not a
`syncOptions` entry).

**Deliberately deferred, not an oversight**: getting ArgoCD to manage its OWN install
(vendoring the install manifest, wiring a self-referential root `Application`) has real
bootstrap-ordering risk — if it goes wrong, the instance that's currently the only thing
running `platform-cicd`'s live pipelines and this very cluster-config repo's own sync
goes with it. Still true, still deferred: version stays pinned as a real vendored
manifest (`install.yaml`) in the meantime, not just a version number in prose.

This used to bundle the two-ArgoCD-instance split in with that same deferral (`gitops-
strategy.md` §4's "Phase 4"). Unbundled 2026-08-16, deliberately: the split itself
carries none of the same risk (see "argocd-apps split" below) since `argocd-apps` is a
brand-new instance with zero live workload on it, so there was no reason to keep waiting
on the higher-risk self-management piece before doing the split.

## `argocd-apps` split — live-verified 2026-08-16

`gitops-strategy.md` §2's two-instance split, built without touching this instance's own
bootstrap: `argocd-apps-install/application.yaml` installs a second `argo-cd` Helm
release (chart `10.2.1`/`v3.4.5` - deliberately matching this instance's own running
version exactly, confirmed live via `kubectl get deployment argocd-server -n argocd -o
jsonpath='{.spec.template.spec.containers[0].image}'`, not the newer `10.3.2` `kind-
prod`'s single-instance install uses - both instances on this cluster share one set of
CRDs, so keeping them version-identical avoids any controller/CRD-schema skew) into its
own `argocd-apps` namespace, `crds.install: false` to reuse the CRDs this directory's own
`install.yaml` already put in the cluster.

**No `application.namespaces` cross-namespace watch configured on either instance,
deliberately, as of this split** (`argocd-platform` later grows a narrow, deliberate
carve-out for tenant `-cicd` namespaces specifically - see "`*-cicd` namespace watch"
below; `argocd-apps` remains untouched) - confirmed live this is what actually delivers
the isolation at the time of this split: each
instance's controllers default-scope to Applications/ApplicationSets/AppProjects
physically inside their own release namespace only. Verified two ways: (1) the 4
per-app-Application-generating objects (`tenant-onboarding`, `tenant-appprojects`,
`xr-requests` ApplicationSets, `idp-onboarding` AppProject) were moved into `argocd-apps`
by editing their `metadata.namespace` in git, and `argocd-platform`'s own root correctly
stopped managing them (the old copies just sat `OutOfSync`/`requiresPruning: true`,
manually deleted once the new copies were confirmed `Synced`+healthy - zero live
generated children existed under the old namespace before this move, so nothing was lost
tearing them down) while `argocd-apps`'s own `applicationset-controller` picked them up
and reconciled all 3 cleanly (`generated 0 applications` - correct, zero real tenants
onboarded yet); (2) a real RBAC boundary test: a throwaway ServiceAccount scoped only to
the `argocd-apps` namespace got a hard `Forbidden` (not just a permissive `auth can-i`
check) attempting to `list secrets` in the `argocd` namespace, confirming this isolation
holds at the real k8s API level, not just by convention.

One real transient issue hit live, not a config bug: `argocd-apps-repo-server` crash-
looped 5 times on first boot (`gpg-wrapper.sh: Cannot fork`, `git fetch ... unable to
create thread`) during the burst of all-new-instance pods starting simultaneously - node
had 20Gi available memory and 160045 PID ulimit headroom at the time (confirmed live via
`podman exec dev-control-plane free -h`/`ulimit -a`), so this wasn't real resource
exhaustion, just contention during the startup burst. Self-healed via Kubernetes' own
crash-loop backoff/retry with zero manual intervention.

## `*-cicd` namespace watch — added live, 2026-08-16

`platform-cicd`'s per-tenant onboarding (`nodejs-demo-app-pr-envs` ApplicationSet, its
GitHub-token Secret, and the refresher CronJob's RBAC - see `platform-cicd`'s
`docs/ephemeral-environments.md`) used to be pinned to the `argocd` namespace like
everything else on this instance, which forced its token Secret there too and needed a
narrowly-scoped but still cross-namespace Role/RoleBinding just to write it. Moved
instead into the app's own `<type>-<app-name>-cicd` namespace, using ArgoCD's
ApplicationSet-in-any-namespace feature:

```
data:
  application.namespaces: "*-cicd"
  applicationsetcontroller.namespaces: "*-cicd"
  applicationsetcontroller.allowed.scm.providers: "https://api.github.com"
  applicationsetcontroller.enable.tokenref.strict.mode: "true"
```

Deliberately scoped to the `*-cicd` glob only, not the broader `app-*`/`infra-*`
destinations `platform-onboarding`'s `AppProject` already uses - `-cicd` is the one
namespace class where "only this chart's own GitOps-managed release writes here" still
holds, so widening `argocd-platform`'s own watch into it doesn't cross into
`argocd-apps`'s territory or grant this instance any capability it didn't already have
(it already owns Namespace/RBAC creation there via `platform-onboarding`). `argocd-apps`'s
own `argocd-cmd-params-cm` is untouched.

The last two keys weren't optional extras, both found live, not anticipated up front:

1. **`applicationsetcontroller.allowed.scm.providers` is required, not optional, the
   moment `applicationsetcontroller.namespaces` is set to anything non-default.**
   `argocd-applicationset-controller` refused to start at all without it -
   `"When enabling applicationset in any namespace using applicationset-namespaces, you
   must either set --enable-scm-providers=false or specify --allowed-scm-providers"` -
   crash-looping (`Error`, exit 1) instead of degrading gracefully. `pullRequest.github`
   defaults to `https://api.github.com` when no `api:` override is set (confirmed against
   ArgoCD's own docs, not guessed), so that's the value listed.
2. **`applicationsetcontroller.enable.tokenref.strict.mode` makes ArgoCD reject any
   `tokenRef` Secret that isn't labeled `argocd.argoproj.io/secret-type: scm-creds`** -
   confirmed live: the controller logged exactly that rejection reason against the
   old (pre-migration) Secret in `argocd`, which never carried the label, the moment
   strict mode came up. Enabled deliberately alongside the namespace widening rather
   than left off, since without it any ApplicationSet living in a `*-cicd` namespace
   could read any Secret in that namespace via `tokenRef`, not just ones meant for it.

Applied both live (`kubectl patch cm argocd-cmd-params-cm -n argocd --type merge -p
'{"data":{...}}'` + `argocd-application-controller`/`argocd-applicationset-controller`
restarts to pick it up) and in `install.yaml` itself, so a from-scratch bootstrap
reproduces it. `argocd-application-controller` restarted clean on the first try;
`argocd-applicationset-controller` needed the SCM-allowlist fix above before its restart
came up healthy.

## `reposerver.repo.cache.expiration` — found live, 2026-08-15

Found chasing an `xr-requests/` deletion investigation
(`idp/docs/service-catalog-design.md` §0): a brand-new `tenants/<app>/` directory sat
at "generated 0 applications" for 15+ minutes, surviving multiple `argocd-repo-server`
pod restarts. Root cause: the `directories`/`files` generators' listing responses are
cached in Redis (`--repo-cache-expiration`, stock default 24h) — a separate, longer-
lived layer than `--revision-cache-expiration` (3m default, just branch-name-to-SHA
resolution). Restarting `argocd-repo-server` only clears its local in-memory git
clone, not this Redis-backed layer, which survives pod restarts; only an
`argocd-redis` restart (or waiting out the full 24h) busted it.

Fixed by setting `reposerver.repo.cache.expiration: 3m` in `argocd-cmd-params-cm` —
matched to `revision-cache-expiration`'s own already-short default rather than
inventing a new interval, and short enough that it can't outlast the git generators'
own ~180s reconcile cycle, so every reconcile now genuinely re-fetches instead of
serving a stale same-cache-key response for up to a day. Live-verified: a throwaway
onboarding commit surfaced as a real `Application` in ~3 minutes with zero manual
cache-busting, versus 15+ minutes (uncertain upper bound) before. Applied both live
(`kubectl patch cm argocd-cmd-params-cm -n argocd --type merge -p
'{"data":{"reposerver.repo.cache.expiration":"3m"}}'` + a `repo-server` restart to
pick it up) and in `install.yaml` itself, so a from-scratch bootstrap reproduces it.
