# etcd-backup

Scheduled OpenShift etcd snapshot CronJob. Runs privileged on a control-plane node, chroots into the host to invoke `cluster-backup.sh`, and stores snapshots on a PVC with directory-based retention.

## Layout

```text
etcd-backup/
├── base/                 # reusable base manifests
└── overlays/
    └── production/       # example production customizations
```

## Apply

```bash
# Labs / defaults
oc apply -k ./base

# Production example (edit SC / hostname / digest first)
oc apply -k ./overlays/production
```

Preview with `kubectl kustomize ./base` or `kubectl kustomize ./overlays/production`.

## Base vs production overlay

| | Base | Production overlay (example) |
|---|------|------------------------------|
| Schedule | `0 1 * * *` | `0 2 * * *` |
| Retention | 5 runs | 14 runs |
| PVC size | 20Gi | 50Gi |
| StorageClass | cluster default | `ocs-storagecluster-ceph-rbd` (edit for your cluster) |
| Resources | 100m/256Mi req, 1 CPU/1Gi lim | 200m/512Mi req, 2 CPU/2Gi lim |
| Job limits | backoff 2, deadline 1h | backoff 3, deadline 2h |

## Before production apply

1. `overlays/production/pvc-patch.yaml` — `storageClassName` and size  
2. `overlays/production/cronjob-patch.yaml` — schedule, retention, optional hostname pin  
3. `overlays/production/kustomization.yaml` — image digest under `images:`

## Custom overlay

```yaml
# overlays/my-cluster/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

patches:
  - target:
      kind: CronJob
      name: etcd-backup
    patch: |-
      - op: replace
        path: /spec/schedule
        value: "0 3 * * *"
```
