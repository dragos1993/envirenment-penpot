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
  general best practice — it's a workaround for this specific CRC VM's
  small default memory allocation. Confirmed directly: even a
  completely empty, fresh CRC VM already has ~96% of memory *requested*
  by OpenShift's own core components alone (`kube-apiserver`, `etcd`,
  monitoring, OLM, image registry, ingress, DNS, ...), before Penpot or
  anything else is deployed — this isn't primarily about other apps
  cluttering the cluster, it's this default VM sizing leaving very
  little headroom, period. See the comment at the top of the file, and
  `release-penpot/README.md`'s "Configuration reference" section, for
  the full CRC-vs-enterprise breakdown and the mechanics of why
  `limits`-only overrides don't work here (a `limits` without a
  matching `requests` gets its `requests` auto-filled to match by the
  API server). `release-penpot/INSTALL.md` has the full incident
  narrative, including growing the VM's memory (`crc config set memory
  12288`) as the fix that actually stopped recurring control-plane
  instability — this values file's BestEffort override stayed in place
  even after that, since 12Gi still leaves only modest headroom.

  **On a properly-sized or enterprise cluster**, none of this file's
  `resources` overrides should exist — delete them (or don't create a
  values file with them at all) and let `release-penpot/values.yaml`'s
  own defaults apply, which give real Burstable-tier QoS instead.

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

## Verifying these overrides actually took effect

For the full "is Penpot actually working" checks (Postgres/Valkey/MinIO
connectivity, HTTP checks), see `release-penpot/README.md`'s
"Verifying it's all actually wired up" section. What's specific to
*this* repo is confirming the values here were actually applied:

```bash
# Hostname from values-dev.yaml's publicHost
oc get route penpot -n penpot -o jsonpath='{.spec.host}{"\n"}'

# QoS class -- should print "BestEffort" for every pod while the
# resources:{} override in values-dev.yaml is in place
oc get pods -n penpot -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.status.qosClass}{"\n"}{end}'

# Storage sizes requested (crc-csi-hostpath-provisioner doesn't
# enforce these, but the PVC spec should still match what was asked for)
oc get pvc -n penpot -o jsonpath='{range .items[*]}{.metadata.name}{" requested="}{.spec.resources.requests.storage}{" capacity="}{.status.capacity.storage}{"\n"}{end}'
```

If you've since removed the `resources:{}` override (e.g. after moving
to a bigger cluster), the QoS class command should print `Burstable`
instead, matching `release-penpot/values.yaml`'s defaults.
