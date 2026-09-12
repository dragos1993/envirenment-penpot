# envirenment-penpot

Environment-specific Helm values for the
[`release-penpot`](https://github.com/dragos1993/release-penpot) chart.

The chart itself carries no environment knowledge (hostnames, storage
sizes, resource sizing) — that lives here instead, one file per
environment, layered on top of the chart's `values.yaml` defaults. This
keeps the chart reusable across environments and keeps environment drift
visible in one small diff-able file per environment, instead of forked
copies of the whole chart.

## Files

- `values-dev.yaml` — the OpenShift Local (CRC) dev/test environment.
  Notably, it clears `resources` (both `requests` and `limits`) on every
  Penpot component, making them BestEffort QoS pods. This is **not** a
  general best practice — it's a workaround for this specific CRC node
  running with very little spare *requested* memory (it already hosts
  ArgoCD, Tekton Pipelines, and another sample app). See the comment at
  the top of the file, and `release-penpot/README.md`'s "Known dev-cluster
  caveat" section, for the mechanics of why `limits`-only overrides don't
  work here (a `limits` without a matching `requests` gets its `requests`
  auto-filled to match by the API server).

## Usage

Directly with Helm:

```bash
helm install penpot ../release-penpot -n penpot -f values-dev.yaml
```

Or via ArgoCD, which references this repo as a second Helm "source" for
its `valueFiles` — see
[`argocd-repo`](https://github.com/dragos1993/argocd-repo)'s
`apps/penpot-app.yaml` and README.

## What does *not* belong here

Real credentials. `penpot-secrets` (Postgres password, MinIO root
credentials, Penpot's session secret key) is created directly in-cluster
with `oc create secret`, never committed to this repo or templated by the
chart — see `release-penpot/README.md` for the exact command.
