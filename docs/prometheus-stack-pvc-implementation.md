# Giving the monitoring stack real storage

Prometheus, Alertmanager and Grafana all run on `emptyDir` right now. The
`prometheus-kube-prometheus-stack-prometheus-db` volume is literally `{}`, and Grafana's `storage`
volume is the same. Every pod restart throws away the metric history and any dashboard that was
edited through the UI — and those pods restarted four times in the last hour, so this isn't a
theoretical problem. It's the persistence entry in [current-issues.md](current-issues.md).

This is the plan for fixing it. Nothing here is applied yet.

Two decisions up front: **30 days of retention**, and **trim the metric cardinality at the same
time**, because the second one changes the answer to the first.

## Why a standalone PVC doesn't work

The obvious move is to write a PersistentVolumeClaim and point the stack at it. That fails for all
three components, for two different reasons.

**Prometheus and Alertmanager** are StatefulSets, and they aren't mine — prometheus-operator
(`v0.93.1`) owns them. Storage has to be declared on the `Prometheus` and `Alertmanager` custom
resources via `storageSpec.volumeClaimTemplate`. The operator renders that into the StatefulSet's
`volumeClaimTemplates` and names the PVCs itself. A PVC I create by hand is something the operator
will never look at; it would sit there doing nothing.

**Grafana** is an ordinary Deployment, and the grafana subchart already knows how to create its own
PVC from `grafana.persistence`. Wiring in an external claim via `existingClaim` is possible but
buys nothing.

So all of this belongs in the `values:` block of
`monitoring/controllers/base/kube-prometheus-stack/release.yaml`, and the draft
`monitoring/controllers/base/kube-prometheus-stack/pvc.yaml` gets deleted. It's untracked, it isn't
referenced by the kustomization, and it's still a copy of the linkding claim right down to
`metadata.name: linkding-data-pvc`.

## How big, and why

Measured against the running cluster rather than guessed:

| Measurement | Value |
|---|---|
| Active head series | 98,318 |
| Scrape interval | 30s |
| Bytes per sample, compressed | 0.91 |
| Scrape targets | 13 |
| Free disk on the node | ~1.8 TiB |
| `local-path` `allowVolumeExpansion` | **false** |

98,318 series over a 30s interval is about 3,280 samples/sec, and at 0.91 bytes each that's
**~258 MB/day** — call it 7.7 GB for 30 days. After the cardinality trim below it drops to roughly
56k series, or **~147 MB/day, about 4.4 GB over 30 days**.

Two properties of `local-path` matter more than the arithmetic:

1. **It can't be expanded.** `allowVolumeExpansion` is false, so whatever size I ask for is fixed
   until I destroy the volume and rebuild it. With 1.8 TiB free, over-provisioning is free and
   under-provisioning costs a rebuild.
2. **It doesn't enforce the request.** local-path is a bind mount into a directory on the host, so
   the `storage:` request is advisory — nothing stops Prometheus filling the entire disk. The real
   guard is `retentionSize`, which is why it's set explicitly rather than left to default.

That gives:

- **Grafana — 2Gi.** Grafana stores a SQLite `grafana.db` holding dashboards, users, annotations
  and sessions, plus the plugin directory. Actual use is single-digit megabytes. 2Gi is about 100x
  headroom and still nothing against this disk. 512Mi would almost certainly survive, but since it
  can't be grown later there's no reason to cut it that fine.
- **Prometheus — 20Gi, `retention: 30d`, `retentionSize: 15GB`.** The retention cap sits roughly 3x
  above projected usage so series growth doesn't hit it, and the volume sits above the cap because
  compaction transiently needs space beyond the blocks it's retaining.
- **Alertmanager — 1Gi.** Silences and the notification log. Kilobytes, realistically.

## The changes

Added to `spec.values`. No `storageClassName` anywhere — inheriting the default `local-path` is how
`apps/base/linkding/pvc.yaml` already does it, and naming it would only make the manifest lie on a
cluster where the default differs.

```yaml
prometheus:
  prometheusSpec:
    retention: 30d
    retentionSize: 15GB
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 20Gi

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 1Gi
```

The existing `grafana:` block gains:

```yaml
  persistence:
    enabled: true
    type: pvc
    accessModes: ["ReadWriteOnce"]
    size: 2Gi
  deploymentStrategy:
    type: Recreate
```

`deploymentStrategy: Recreate` is doing real work there. The default rolling update starts the
replacement pod before terminating the old one, and both would want the same ReadWriteOnce volume.
On a single node that happens to be fine — RWO is enforced per node, not per pod — but it breaks
the moment there's a second node, which is exactly where this cluster is heading.

## Cutting the cardinality

98k series from 13 targets is a lot, and it's concentrated in histogram buckets on the apiserver
scrape job:

| Metric | Series |
|---|---|
| `etcd_request_duration_seconds_bucket` | 8,976 |
| `apiserver_request_duration_seconds_bucket` | 8,704 |
| `apiserver_watch_list_duration_seconds_bucket` | 4,572 |
| `apiserver_request_body_size_bytes_bucket` | 3,328 |
| `apiserver_request_sli_duration_seconds_bucket` | 3,240 |
| `apiserver_watch_cache_read_wait_seconds_bucket` | 3,220 |
| `workqueue_work_duration_seconds_bucket` + `workqueue_queue_duration_seconds_bucket` | 4,312 |
| `apiserver_response_sizes_bucket` | 2,096 |
| `apiserver_watch_events_sizes_bucket` | 2,070 |
| `apiserver_admission_controller_admission_duration_seconds_bucket` | 826 |
| `apiserver_flowcontrol_request_execution_seconds_bucket` | 504 |

That's ~41.8k series, about 43% of the total, and it exists to power apiserver latency SLO
dashboards that nobody opens on a single-node homelab. Dropped at scrape time:

```yaml
kubeApiServer:
  serviceMonitor:
    metricRelabelings:
      - sourceLabels: [__name__]
        action: drop
        regex: '(apiserver_request_duration_seconds_bucket|apiserver_watch_list_duration_seconds_bucket|apiserver_request_body_size_bytes_bucket|apiserver_request_sli_duration_seconds_bucket|apiserver_watch_cache_read_wait_seconds_bucket|apiserver_response_sizes_bucket|apiserver_watch_events_sizes_bucket|apiserver_admission_controller_admission_duration_seconds_bucket|apiserver_flowcontrol_request_execution_seconds_bucket|etcd_request_duration_seconds_bucket|workqueue_work_duration_seconds_bucket|workqueue_queue_duration_seconds_bucket)'
```

Dropping metrics has a consequence that has to be handled in the same commit. The chart ships
recording rules and alerts built on exactly these buckets. Left enabled, they quietly evaluate to
nothing, or fire because the data they expect has vanished — which is worse than not having them,
because an alert that's always broken is an alert I stop reading. So the dependent groups go off
too:

```yaml
defaultRules:
  rules:
    kubeApiserverAvailability: false
    kubeApiserverBurnrate: false
    kubeApiserverHistogram: false
    kubeApiserverSlos: false
    etcd: false
```

One trap when applying this: `kubeApiServer.serviceMonitor.metricRelabelings` **replaces** the
chart's default list rather than appending to it. Render the chart and diff the resulting
ServiceMonitor against the one currently in the cluster before committing, so nothing the chart was
already dropping gets quietly re-enabled.

The kubelet job is actually the biggest single source at 39,525 series, mostly cAdvisor. I'm
leaving it alone: it's spread thinly across many metric names rather than concentrated in a few fat
ones, so trimming it is fiddly, and per-container CPU and memory is the data I'm most likely to
actually want.

## Applying it

`volumeClaimTemplates` is immutable on a StatefulSet, and both StatefulSets currently have none —
so this isn't an edit, it's a replacement. Operator v0.93.1 detects the rejected update and
recreates the StatefulSet itself, so it should converge without help. If the HelmRelease goes
`Failed` with a StatefulSet update error, the fallback is to delete it and let the operator rebuild:

```
kubectl delete sts -n monitoring prometheus-kube-prometheus-stack-prometheus --cascade=orphan
```

There's nothing to lose either way. It's all `emptyDir` today.

## Checking it worked

1. `kubectl get pvc -n monitoring` — three Bound claims: Prometheus 20Gi, Alertmanager 1Gi,
   Grafana 2Gi.
2. The volume is no longer ephemeral:
   ```
   kubectl get sts -n monitoring prometheus-kube-prometheus-stack-prometheus \
     -o jsonpath='{.spec.volumeClaimTemplates[*].spec.resources.requests.storage}'
   ```
3. Retention took effect — expect `30d 15GB`:
   ```
   kubectl get prometheus -n monitoring \
     -o jsonpath='{.items[0].spec.retention} {.items[0].spec.retentionSize}'
   ```
4. Cardinality is down, using the same query the numbers above came from. Expect roughly 56k,
   against 98,318 before:
   ```
   kubectl get --raw "/api/v1/namespaces/monitoring/services/kube-prometheus-stack-prometheus:9090/proxy/api/v1/query?query=prometheus_tsdb_head_series"
   ```
5. **The restart test**, which is the entire point. Delete the Prometheus pod, wait for it to come
   back, and confirm data from before the restart is still queryable.
6. `flux get kustomizations` and `flux get helmreleases -n monitoring` both Ready, and no rules
   left referencing dropped metrics:
   ```
   kubectl get prometheusrules -n monitoring -o yaml | grep -c apiserver_request_duration_seconds_bucket
   ```
   should print `0`.

## While I'm in this file

`grafana.adminPassword: adi` is still sitting in cleartext in the same `release.yaml` this change
edits. Tempting to fix in passing, but it needs an actual password rotation rather than just an
edit — the committed value has to be treated as burned. Tracked separately in
[current-issues.md](current-issues.md).
