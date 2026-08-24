# 30-policy

Cluster-wide guardrails via Kyverno — real as of 2026-08-24.

- `kyverno/` — the engine (Helm chart, pinned 3.9.0 / appVersion v1.19.0, the first
  release with the `ValidatingPolicy` CRD generally available).
- `kyverno-policies/` — plain-manifest `ValidatingPolicy` objects. First one:
  `testkube-secret-usage.yaml`, closing a real cross-tenant secret-reference gap in
  Testkube's shared `testkube` namespace — see that file's own header and
  platform-cicd's ADR-0007/ADR-0008 for the full story.

Gated by `components.policy` in `cluster.yaml` (see `cluster.yaml.example`) - only
meaningful today alongside `components.platformCicd: true`, since the one shipped
policy governs Testkube TestWorkflow secret usage.
