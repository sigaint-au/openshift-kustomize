# network-policy

You open exactly what your apps need inside a locked-down project.
Copy this directory into your app repo and apply it with your
workloads. It pairs with the
[baseline-admin-network-policy](../baseline-admin-network-policy/)
baseline, which denies pod traffic until these policies open it.

## What you get

```text
network-policy/
├── kustomization.yaml            # namespace: my-app, change to your project
├── allow-from-same-namespace.yaml
├── allow-from-ingress.yaml
├── allow-from-monitoring.yaml
└── allow-dns-egress.yaml
```

| Policy | Opens |
|--------|-------|
| `allow-from-same-namespace` | Ingress and egress between pods in your project |
| `allow-from-ingress` | Ingress from the OpenShift router |
| `allow-from-monitoring` | Ingress from cluster Prometheus for scrapes |
| `allow-dns-egress` | Egress to cluster DNS on port 53 |

You need the DNS policy. The baseline denies pod-to-pod egress, and
cluster DNS runs as pods, so your apps lose name resolution without it.

## Requirements

You need nothing beyond a project and the standard
`networking.k8s.io/v1` API. No cluster admin, no operator.

## Use it

Copy this directory into your app repo. Set `namespace` in its
`kustomization.yaml` to your project name. Apply it with your workloads:

```bash
oc apply -k ./network-policy
```

Check what landed:

```bash
oc get networkpolicy -n my-app
```

You should see all four policies. Test four paths: pod to pod in the
project, a Route through the router, a metrics scrape, and a DNS lookup
from a pod.

## Restrict router traffic to app ports

The template opens all pod ports so it works on drop-in. Once your
app runs, pin `allow-from-ingress` to its container ports:

```yaml
# allow-from-ingress.yaml
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: openshift-ingress
      ports:
        - protocol: TCP
          port: 8080
        - protocol: TCP
          port: 8443
```

Apply and re-test the Route. Do the same for `allow-from-monitoring`
with your metrics port once you know it. Keep your container ports in
sync with these blocks.

## Close outside egress too

These policies govern pod-to-pod traffic only. For addresses outside
the cluster, add the [egress-firewall](../egress-firewall/) snippet
beside this one and apply both together.

## Remove

```bash
oc delete -k ./network-policy
```

Deleting the policies closes what they opened. The baseline deny, if
your project is enrolled, takes full effect again.
