# Current issues

Known problems with the cluster as it stands. I'd rather write these down than have someone
find them for me. Roughly ordered by how much they'd cost if they bit.

## Plaintext Grafana admin password

`monitoring/controllers/base/kube-prometheus-stack/release.yaml` sets `grafana.adminPassword`
as a literal string in the HelmRelease values. It's committed in a public repo, and rewriting
history won't help once it's been pushed.

**Fix:** create a SOPS-encrypted Secret under `monitoring/configs/staging/` and point the chart
at it with `grafana.admin.existingSecret` plus `userKey`/`passwordKey`, then rotate the password
itself — the old one has to be treated as burned regardless of what the repo says afterwards.
The plumbing already exists; `grafana-tls-secret` is decrypted by the same Kustomization.

## Misspelled securityContext key

`apps/base/linkding/deployment.yaml` has `allowPriviledgeEscalation: false` — an extra `d`.
The correct key is `allowPrivilegeEscalation`. Kubernetes prunes the unknown field on the way in,
so the setting silently does nothing and the container runs with privilege escalation allowed.

This is the argument for CI (below): a schema check against the API would have caught it, and
nothing else was ever going to.

**Fix:** correct the spelling. Worth adding `readOnlyRootFilesystem` and an explicit
`capabilities.drop: [ALL]` while I'm in there, since the intent was clearly to harden the pod.

## No dependency ordering between Kustomizations

Every `dependsOn` block in `clusters/staging/` is commented out, along with the `infra-configs`
layer they refer to — which doesn't exist yet either. So `apps`, `monitoring-controllers` and
`monitoring-configs` all reconcile in parallel with no ordering guarantee.

The visible symptom is on a cold cluster: `monitoring-configs` tries to create `grafana-tls-secret`
in the `monitoring` namespace before `monitoring-controllers` has created that namespace. Flux
retries on a 1m interval and it resolves itself within a couple of reconciles, so the cluster
converges — it just converges noisily, and a transient failure that's expected is a failure I'll
stop reading.

**Fix:** `dependsOn: [{name: monitoring-controllers}]` on `monitoring-configs` is the immediate
one. The larger version is building the `infrastructure/` layer the commented blocks anticipate,
so that ingress and certificate machinery is a real dependency edge rather than a race that
happens to settle.

## No persistence for Prometheus or Grafana

The HelmRelease customises Grafana's ingress and nothing else, so Prometheus, Alertmanager and
Grafana all run on chart defaults — which means no `storageSpec` on the Prometheus StatefulSet
and an `emptyDir` for Grafana. A pod restart loses all metric history and any dashboard edited
through the UI.

It hasn't hurt yet because I mostly look at live data, but it makes the monitoring stack useless
for the thing monitoring is actually for: working out what happened before a failure I noticed
after the fact.

**Fix:** set `prometheus.prometheusSpec.storageSpec` with a PVC template and `grafana.persistence`.
On a single node with `local-path` that's straightforward. It gets more interesting once there's
more than one node and the volume stops being reachable from wherever the pod lands — see the
multi-node section of the roadmap.

## Implicit kustomization in apps/staging

The `apps` Kustomization points at `./apps/staging`, and that directory has no
`kustomization.yaml`. Flux's kustomize-controller falls back to generating one by scanning the
directory for manifests, which is why it works.

It works, but it's the one place in the repo where what gets applied isn't written down. The
`.sops.yaml` sitting in that directory gets swept up in the scan too. Anything dropped into that
folder is deployed by virtue of existing, which isn't the property I want from a GitOps repo.

**Fix:** add an explicit `kustomization.yaml` listing `linkding/` as its only resource.

## No CI

There's no `.github/workflows`, no pre-commit config, and no Flux `Alert`/`Provider` wired up
even though notification-controller is installed. Nothing validates a manifest before Flux tries
to apply it, and nothing tells me when a reconciliation fails — I find out by running `flux get
kustomizations` or by noticing a service is down.

**Fix:** two separate pieces.
- A workflow running `kustomize build` over each overlay and `kubeconform` against the real CRD
  schemas, so structural mistakes fail on the PR rather than in the cluster.
- A notification-controller `Provider` and `Alert` pushing reconciliation failures somewhere I'll
  actually see them.
