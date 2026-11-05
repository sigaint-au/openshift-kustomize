# stackrox-security-policy

You deploy a sensible baseline of StackRox / RHACS security policies
via kustomize and GitOps. Each policy is a separate file — drop the
ones you don't need, add enforcement after you've tested in inform
mode.

Policies apply to every deployment in every cluster Central monitors.
Use exclusions (in the YAML or RHACS UI) for workloads that
legitimately need elevated access.

## What you get

```text
stackrox-security-policy/
├── kustomization.yaml            # namespace: stackrox, change if Central lives elsewhere
├── privileged-containers.yaml    # Block securityContext.privileged
├── run-as-root.yaml              # Require non-root USER directive
├── host-network.yaml             # Block hostNetwork: true
├── latest-image-tag.yaml         # Block 'latest' image tag
├── container-runtime-socket.yaml # Block docker/crio socket mounts
├── sensitive-host-mounts.yaml    # Block mount of /etc /sys /dev /proc /var
├── read-only-root-fs.yaml        # Require read-only root fs (disabled by default)
├── unscanned-images.yaml         # Require image scans (disabled by default)
├── cap-sys-admin.yaml            # Block CAP_SYS_ADMIN capability
├── privilege-escalation.yaml     # Block allowPrivilegeEscalation: true
├── resource-limits.yaml          # Require CPU request and memory limit
└── mount-propagation.yaml        # Block bidirectional mount propagation
```

## Requirements

- RHACS 4.6+ with the `config.stackrox.io/v1alpha1` CRD.
- Cluster admin to apply the CRD if it wasn't installed by the Helm
  chart or operator.
- The policies apply to Central's namespace (usually `stackrox`).
  The RHACS config controller lives there and reconciles policy CRs.

One `kubectl explain securitypolicy.spec` gives you every available
field.

## Use it

Copy this directory into your GitOps repo. Set the namespace in
`kustomization.yaml` to wherever Central is installed. Then apply:

```bash
kubectl apply -k ./stackrox-security-policy
# or with OpenShift:
oc apply -k ./stackrox-security-policy
```

Check what landed:

```bash
oc get securitypolicies -n stackrox
```

Log into the RHACS portal → Policy Management. Each policy shows
"Active" and fires violations in inform mode (no enforcement). Before
turning on enforcement, let violations accumulate for a week and
review what gets flagged.

Policies disabled by default (`read-only-root-fs`, `unscanned-images`)
won't fire. Enable them after reviewing the violations they'd produce
in your environment.

### Add enforcement

Edit a policy file and add `enforcementActions`:

```yaml
spec:
  enforcementActions:
    - SCALE_TO_ZERO_ENFORCEMENT   # scales replicas to 0 on violation
```

Deploy-time enforcement options:

| Action | Effect |
|--------|--------|
| `SCALE_TO_ZERO_ENFORCEMENT` | Scales replicas to 0 |
| `FAIL_KUBE_REQUEST_ENFORCEMENT` | Blocks via admission webhook |
| `FAIL_BUILD_ENFORCEMENT` | Fails CI/CD pipeline at `roxctl image check` |

Runtime enforcement: `KILL_POD_ENFORCEMENT`.

Start with `SCALE_TO_ZERO_ENFORCEMENT` in a dev cluster before going
to `FAIL_KUBE_REQUEST_ENFORCEMENT` in production.

### Exclude deployments

Add exclusions inline in the policy YAML under `spec.exclusions`:

```yaml
spec:
  exclusions:
    - name: My CI runner needs the docker socket
      deployment:
        name: ci-runner
        scope:
          namespace: ci
```

All shipped policies already exclude system namespaces
(`kube.*`, `openshift.*`, `redhat.*`, `istio-system`, `gitops-pipelines`)
via a single exclusion regex.

### Remove a policy

Remove its line from `kustomization.yaml` and re-apply. Or delete the
resource:

```bash
oc delete securitypolicy -n stackrox privileged-containers
```

The policy is removed from the RHACS database on the next
reconciliation cycle (up to 10 minutes).

## Common policies

The shipped set covers the most impactful deploy-time checks from the
Red Hat default-enabled baseline. Below are additional policies you'll
commonly want, with their field names for writing new CRs.

| Policy | fieldName | lifecycleStages | severity |
|--------|-----------|----------------|----------|
| Privileged Container | `Privileged Container` | DEPLOY | HIGH |
| Root user in image | `Image User` with `0` or `root` | BUILD, DEPLOY | MEDIUM |
| Host network | `Host Network` | DEPLOY | MEDIUM |
| Host PID | `Host PID` | DEPLOY | MEDIUM |
| Host IPC | `Host IPC` | DEPLOY | MEDIUM |
| Latest image tag | `Image Tag` with `latest` | BUILD, DEPLOY | LOW |
| Container runtime socket | `Volume Source` with socket paths | DEPLOY | HIGH |
| Sensitive host directories | `Volume Source` with directory regex | DEPLOY | MEDIUM |
| Read-only root filesystem | `Read-Only Root Filesystem` | DEPLOY | MEDIUM |
| Unscanned image | `Unscanned Image` | DEPLOY | MEDIUM |
| CAP_SYS_ADMIN | `Add Capabilities` with `SYS_ADMIN` | DEPLOY | MEDIUM |
| Allow privilege escalation | `Allow Privilege Escalation` | DEPLOY | MEDIUM |
| Drop all capabilities | `Drop Capabilities` with `ALL` (negate: true) | DEPLOY | LOW |
| Automount SA token | `Automount Service Account Token` + `Service Account` `default` | DEPLOY | MEDIUM |
| Missing ingress network policy | `Has Ingress Network Policy` | DEPLOY | MEDIUM |
| Missing egress network policy | `Has Egress Network Policy` | DEPLOY | MEDIUM |
| Exposed Node Port | `Exposed Node Port` | DEPLOY | MEDIUM |
| Container CPU limit | `Container CPU Limit` | DEPLOY | LOW |
| Container memory limit | `Container Memory Limit` | DEPLOY | LOW |
| Mount propagation (bidirectional) | `Mount Propagation` with `BIDIRECTIONAL` | DEPLOY | MEDIUM |

To write a new policy CR, use the shipped files as templates and run
`oc explain securitypolicy.spec` for field reference.

## Pair with

- [rhacs-operator](https://docs.redhat.com/documentation/red_hat_advanced_cluster_security_for_kubernetes/) for lifecycle management of Central itself
- [network-policy](../network-policy/) for per-project traffic rules
- [egress-firewall](../egress-firewall/) for outside-cluster egress control

## Remove

```bash
oc delete -k ./stackrox-security-policy
```

Deleting the CRs removes the policies from Central. Alerts stop, and
enforcement pauses, on the next reconciliation cycle.