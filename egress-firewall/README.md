# egress-firewall

You control which outside addresses your project pods can reach. The
shipped object allows your registry and one outside API, then denies
the rest.

## What you get

```text
egress-firewall/
├── kustomization.yaml  # namespace: my-app, change to your project
└── egress-firewall.yaml
```

## Requirements

You need OpenShift with the OVN-Kubernetes plugin. One
`EgressFirewall` named `default` can exist per project. A second
object or another name is rejected, so merge extra rules into this
file instead of adding files.

## Use it

Copy this directory into your app repo. Set `namespace` in its
`kustomization.yaml` to your project name. List your Allow rules
before the closing Deny, then apply it with your workloads:

```bash
oc apply -k ./egress-firewall
oc get egressfirewall -n my-app
```

The shipped object allows `quay.io`, allows HTTPS to one outside API,
and denies the rest. Replace both with the registries and APIs your
pods call. Find your API server range with
`oc get ep kubernetes -n default` and allow it, or the final Deny
locks you out of the cluster API too.

## Test it

Prove each path from a throwaway pod in the project:

```bash
# Allowed registry resolves and connects.
oc -n my-app run netcheck --rm -i --restart=Never \
  --image=registry.redhat.io/rhel9/support-tools \
  -- curl -s -o /dev/null -w "%{http_code}\n" https://quay.io/v2/
# Expect: 200 (or a registry auth challenge, both prove connectivity)

# Anything unlisted fails.
oc -n my-app run netcheck --rm -i --restart=Never \
  --image=registry.redhat.io/rhel9/support-tools \
  -- curl -s -o /dev/null -w "%{http_code}\n" https://unlisted.example.com/
# Expect: curl exits non-zero after a timeout.
```

If your pods use an outside DNS server instead of cluster DNS, allow
its IP addresses in the firewall or name resolution breaks alongside
everything else.

## Pair with

This firewall covers addresses outside the cluster only. For pod to
pod traffic, enroll your project in the
[baseline-admin-network-policy](../baseline-admin-network-policy/)
baseline and open paths with the
[network-policy](../network-policy/) snippet. The baseline README
shows an app repo layout that ships all three together.

## Remove

```bash
oc delete -k ./egress-firewall
```

Deleting the object re-opens outside addresses at once.
