# OpenShift Kustomize Snippets

You get reusable Kustomize bases and overlays for common OpenShift
platform tasks. You can drop each snippet into a production cluster.

## Snippets

| Snippet | What it does |
|---------|----------------|
| [etcd-backup](./etcd-backup/) | CronJob that snapshots etcd on a control-plane node and keeps recent copies on a PVC |
| [baseline-admin-network-policy](./baseline-admin-network-policy/) | BaselineAdminNetworkPolicy that denies pod traffic by default in enrolled projects |
| [network-policy](./network-policy/) | Project NetworkPolicy template that opens same-namespace, router, monitoring, and DNS |
| [egress-firewall](./egress-firewall/) | EgressFirewall that allows your registry and APIs, then denies the rest of outside egress |

See each snippet's README for apply steps, overlays, and cluster requirements.
