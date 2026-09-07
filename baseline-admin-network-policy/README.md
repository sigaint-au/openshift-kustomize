# baseline-admin-network-policy

You lock down pod traffic cluster-wide with one baseline object. You open
only what your apps need with small per-project policies.

## What you get

```text
baseline-admin-network-policy/
├── base/                  # BaselineAdminNetworkPolicy, default deny
└── overlays/
    └── production/        # base plus audit logging
```

The base object denies all pod-to-pod ingress and egress in namespaces
you label `network-isolation=strict`. Your per-project NetworkPolicy
objects override the baseline, so each project opens its own traffic.
Those policies live in the [network-policy](../network-policy/)
snippet: same-namespace, router, monitoring, and DNS allows you copy
into your app repo.

## Requirements

You need OpenShift with the OVN-Kubernetes plugin and the
`policy.networking.k8s.io/v1alpha1` API. That API ships with OpenShift
4.14 and later. You apply the baseline once per cluster as a cluster
admin. You need no StorageClass, no image pull, no extra operator.

One `BaselineAdminNetworkPolicy` named `default` can exist per
cluster. If your cluster has one, merge these rules into it
instead of applying this base.

## Apply the baseline

```bash
# Preview first
kubectl kustomize ./baseline-admin-network-policy/base

# Labs
oc apply -k ./baseline-admin-network-policy/base

# Production, with audit logging of allow and deny verdicts
oc apply -k ./baseline-admin-network-policy/overlays/production
```

Confirm it:

```bash
oc get baselineadminnetworkpolicy default -o yaml
```

## Enroll a project

The baseline touches only namespaces you label. Label yours:

```bash
oc label namespace my-app network-isolation=strict
```

Unlabeled namespaces see no change, including `openshift-*` system
namespaces. Roll the label out one project at a time and watch your
app before you move to the next.

## Open traffic with the network-policy snippet

The baseline alone leaves enrolled projects silent. Copy the
[network-policy](../network-policy/) snippet into your app repo and
apply it with your workloads. It holds the four allows (same-namespace,
router, monitoring, DNS) plus test and port-restriction examples.

## Examples

### Lock down a new project end to end

Start a project with a running app. This session uses `my-app` and a
Service named `hello` on port 8080. Swap in your names.

```bash
# 1. Create the project and deploy your app first, so it runs
#    before any deny rule touches it.
oc new-project my-app
oc apply -f ./my-workloads/

# 2. Enroll the project in the baseline.
oc label namespace my-app network-isolation=strict

# 3. Copy the template next to your workloads and point it at
#    your project, then apply it.
cp -r ./network-policy ./my-workloads/network-policy
# Edit my-workloads/network-policy/kustomization.yaml: set namespace: my-app
oc apply -k ./my-workloads/network-policy
```

Prove each path from a throwaway pod in the project:

```bash
# DNS resolves.
oc -n my-app run netcheck --rm -i --restart=Never \
  --image=registry.redhat.io/rhel9/support-tools \
  -- getent hosts kubernetes.default
# Expect an IP line, for example: 172.30.0.1 kubernetes.default

# Pod to pod works.
oc -n my-app run netcheck --rm -i --restart=Never \
  --image=registry.redhat.io/rhel9/support-tools \
  -- curl -s -o /dev/null -w "%{http_code}\n" http://hello:8080/healthz
# Expect: 200

# Pod to outside pods fails. Run the same curl against a pod IP in
# another enrolled project.
# Expect: curl exits non-zero after a timeout.
```

Then check the Route in a browser or with curl from your machine, and
confirm your ServiceMonitor or PodMonitor target shows Up in the
cluster console. If the Route fails, you missed `allow-from-ingress`.
If scrapes stay Down, you missed `allow-from-monitoring`. If DNS
fails, you missed `allow-dns-egress`.

### Pin ports once the app runs

The template opens all pod ports so it works on drop-in. The
[network-policy](../network-policy/) README shows how to pin
`allow-from-ingress` and `allow-from-monitoring` to your container
ports after that.

### Ship the template with your app repo

Keep policies beside the workloads they protect so every deploy
carries them:

```text
my-workloads/
├── kustomization.yaml   # lists deployment.yaml, service.yaml,
│                        # route.yaml, network-policy/, egress-firewall/
├── deployment.yaml
├── service.yaml
├── route.yaml
├── network-policy/      # copied from ../network-policy/
│   ├── kustomization.yaml  # namespace: my-app
│   ├── allow-from-same-namespace.yaml
│   ├── allow-from-ingress.yaml
│   ├── allow-from-monitoring.yaml
│   └── allow-dns-egress.yaml
└── egress-firewall/     # copied from ../egress-firewall/
    ├── kustomization.yaml  # namespace: my-app
    └── egress-firewall.yaml
```

```yaml
# my-workloads/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: my-app

resources:
  - deployment.yaml
  - service.yaml
  - route.yaml
  - network-policy/
  - egress-firewall/
```

One command deploys app and policy together:

```bash
oc apply -k ./my-workloads
```

## Base vs production overlay

| | Base | Production overlay |
|---|---|---|
| Deny ingress | yes | yes |
| Deny pod egress | yes | yes |
| Audit logging (`acllogging`) | off | allow and deny verdicts logged |
| `environment` label | absent | `production` |

## Control external egress

The baseline has no rule for peers outside the cluster, so traffic to
outside addresses stays open. Close it with the
[egress-firewall](../egress-firewall/) snippet: copy it into your app
repo beside the NetworkPolicy template and apply both together.

## What this does not cover

Source-address control for outbound traffic stays out of scope. If
your security team needs fixed egress IPs per project, add EgressIP
on top of this firewall.

An `AdminNetworkPolicy` deny would override your project policies.
This repo ships only the baseline form, which sits below them, so
projects keep control of their own allows.

## Remove

```bash
oc delete -k ./baseline-admin-network-policy/overlays/production
oc label namespace my-app network-isolation-
```

Deleting the baseline object lifts the deny rules at once. Project
policies stay in place and keep enforcing their own allows and denies.
