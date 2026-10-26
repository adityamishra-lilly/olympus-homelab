# Olympus Homelab

Everything running on my home Kubernetes cluster, declared here. It's a single k3s node that pulls from this repo. I don't `kubectl apply` anything by hand, and if the cluster drifts from what's on `main`, Flux puts it back. Services are published under `am-labs.dev`.

## The hardware

One box: a 2013 **HP ProLiant ML310e Gen8** tower. It's thirteen years old.

## How it reconciles

Flux v2.9.5 is bootstrapped against this repo over an SSH deploy key. It watches `main` and owns one path; everything else is declared from there.

```
GitRepository/flux-system  --  ssh://git@github.com/adityamishra-lilly/olympus-homelab  (main, 1m)
└── Kustomization/flux-system                 ./clusters/staging                10m
    ├── Kustomization/apps                    ./apps/staging                     1m   + SOPS
    ├── Kustomization/monitoring-controllers  ./monitoring/controllers/staging   1m
    └── Kustomization/monitoring-configs      ./monitoring/configs/staging       1m   + SOPS
```

`clusters/staging/` is the only path Flux is told about at bootstrap. Adding a workload means committing a `Kustomization` there, not re-running `flux bootstrap`. Everything below it uses a
base/overlay split: `base/` holds manifests that don't care where they run, `staging/` patches in the environment. There's only one environment today; the structure is there so that a second one
is a directory rather than a refactor.

Monitoring is split one level further, `controllers/` from `configs/`. The Helm machinery and the per-environment secrets reconcile independently, so I can rotate a certificate without touching
the release.

## Layout

```
apps/
  base/linkding/       app manifests, environment-agnostic
  staging/linkding/    overlay plus the encrypted TLS secret
clusters/
  staging/             what Flux reconciles: flux-system/, apps.yaml, monitoring.yaml
monitoring/
  controllers/         HelmRepository + HelmRelease for kube-prometheus-stack
  configs/             per-environment config and secrets for it
```

## What's running

| Workload                     | Namespace    | Managed by         | Reachable at          |
| ---------------------------- | ------------ | ------------------ | --------------------- |
| linkding `1.31.0`            | `linkding`   | Kustomize overlay  | `ld.am-labs.dev`      |
| kube-prometheus-stack `90.x` | `monitoring` | Flux `HelmRelease` | `grafana.am-labs.dev` |
| cloudflared                  | `linkding`   | disabled           | —                     |

**linkding** is a bookmark manager. It runs as UID/GID 33 with a 512Mi `ReadWriteOnce` PVC on the `local-path` provisioner.

**kube-prometheus-stack** arrives as a `HelmRelease` pinned to `90.x`, so Flux takes minor upgrades on its own and I only get involved for a major. CRDs upgrade with `CreateReplace`. Drift detection
is enabled, with a single ignore rule for `/metadata/annotations/prometheus-operator-validated` - the operator stamps that annotation onto `PrometheusRule` objects after install, so without the
exception Flux reads it as drift and fights the operator on every reconcile.

**cloudflared** is how this was first exposed: a tunnel out to `mybookmarks.am-labs.dev`.

## Secrets

SOPS with an age key. The `.sops.yaml` rule encrypts only `data` and `stringData`, so a secret's name, namespace and type stay readable. Encrypted blobs are unreviewable, and I want a diff to show me that I changed the secret I meant to.

The age public key is committed: that's the point of asymmetric encryption. The private half lives in a `sops-age` Secret in `flux-system`, created once by hand and never in git. Flux's kustomize-controller decrypts at reconcile time via `decryption.provider: sops` on the Kustomizations that need it.

## Ingress and TLS

Traefik handles ingress and so does the `local-path` StorageClass. I use plain `networking.k8s.io/v1` Ingress rather than Traefik's `IngressRoute` CRDs, so swapping controllers stays a one-line change.

Certificates are the weakest part right now. They're issued out-of-band and committed as SOPS-encrypted `kubernetes.io/tls` secrets, which makes renewal a manual job I will eventually forget to do. cert-manager with an ACME issuer is the obvious fix.

How the Cloudflare side fits together (SSL modes, Origin CA certs, locking down the origin) is written up in [`docs/cloudflare-tls.md`](docs/cloudflare-tls.md).

## Where this is going

**Multi-node.** On one box the interesting parts of Kubernetes are decorative. PodDisruptionBudgets, anti-affinity, topology spread constraints, draining a node for a kernel update; all of it is a
no-op when there's nowhere else to schedule, so none of it is really being tested. I have a couple of old Mac Minis sitting unused, quiet and low-power enough to leave running, which makes them the
cheapest route to a real control-plane-and-workers layout where those constraints do something.

**HashiCorp Vault.** SOPS handles static material well, but every rotation is a commit and the old value stays in the history forever. Vault behind External Secrets moves this to leased, dynamic credentials and takes secret lifecycle out of the repo.

**Flux image automation.** The image reflector and automation controllers aren't installed, so tags like `sissbruecker/linkding:1.31.0` get bumped by hand. Adding them lets Flux watch the registry and commit the bump itself the pin stays in git, I'm just no longer the one typing it.

**Istio.** More than two workloads need, and that's deliberate. I want mTLS between services and the traffic-level view that metrics alone don't give me, and mostly I want to learn how it fails somewhere that failing is free.

Longer write-ups for each of these live in [`docs/`](docs/).

What is currently broken or half-finished is tracked in [`docs/current-issues.md`](docs/current-issues.md).
