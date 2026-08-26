# 60-backstage

`kind-man`-only: the fleet's developer portal — upstream Backstage (not Red Hat
Developer Hub, a licensing call made explicitly, see `idp/docs/backstage-design.md`),
built as its own image and onboarded onto `platform-cicd` (`kind-dev`) as an `appType:
infra` app rather than through this template. This directory only carries what's
hand-deployed straight to `kind-man`.

- `postgres/` — Bitnami's `postgresql` chart (OCI, `oci://registry-1.docker.io/
  bitnamicharts/postgresql`, pinned `18.8.13`), standalone, real as of 2026-08-26.
  Backstage's own catalog/scaffolder/auth state lives here. `auth.existingSecret`
  works cleanly on this chart (unlike Infisical's bundled subchart, see that
  Application's own header for why that one couldn't) — create the credentials Secret
  by hand before this Application first syncs, same "never pasted to an assistant,
  never committed" convention as everywhere else in this platform:
  ```
  kubectl create namespace backstage
  kubectl create secret generic backstage-postgres-credentials -n backstage \
    --from-literal=postgres-password="$(openssl rand -base64 24)" \
    --from-literal=password="$(openssl rand -base64 24)"
  ```
- `backstage/` — not yet added. Waiting on the Backstage source repo's first real
  image (`ghcr.io/jfillman/backstage`) to exist before writing a Deployment that would
  otherwise just crashloop on a nonexistent tag. See `idp/docs/backstage-design.md`'s
  rollout phases.

Gated by `components.backstage` in `cluster.yaml` (see `cluster.yaml.example`) -
`kind-man` is the only cluster that should ever set this `true`, matching the singleton
pattern `components.secrets.infisicalHost`/`components.platformCicd` already use for
"exactly one cluster in the fleet runs this."
