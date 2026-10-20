# external-secrets-store

You run an ACM (Advanced Cluster Management) hub and you want managed
clusters to pull secrets from the hub instead of storing them locally.
This snippet sets up the hub-side RBAC and the managed-side
ClusterSecretStore so the [External Secrets Operator][eso] (ESO) can
read hub secrets.

## Design

```
┌─ Hub cluster ──────────────────────────────────────┐
│                                                     │
│  Namespace: eso-cluster-001a                        │
│  ┌────────────────────────────────┐                 │
│  │ ServiceAccount: eso-reader     │                 │
│  │ Role: eso-reader               │  get, list,    │
│  │   (secrets)                    │  watch only    │
│  │   (selfsubjectrulesreviews)    │  create only   │
│  │   (selfsubjectaccessreviews)   │  create only   │
│  │ RoleBinding: eso-reader        │                 │
│  └────────────────────────────────┘                 │
│                                                     │
│  Secrets for cluster-001a live here                 │
│  (e.g. registry pull secrets, DB creds, etc.)       │
└─────────────────────┬───────────────────────────────┘
                      │  bearer token + API call
                      │  (read only)
┌─ Managed cluster ───▼───────────────────────────────┐
│                                                     │
│  Namespace: external-secrets                        │
│  ┌────────────────────────────────┐                 │
│  │ Secret: hub-token              │  bearer token   │
│  └────────────────────────────────┘                 │
│  ┌────────────────────────────────┐                 │
│  │ ClusterSecretStore: hub-store   │                 │
│  │   provider: kubernetes          │                 │
│  │   remoteNamespace: eso-cluster-001a              │
│  └────────────────────────────────┘                 │
│  ┌────────────────────────────────┐                 │
│  │ ExternalSecret: example-secret  │                 │
│  │   pulls "my-secret"            │                 │
│  │   → creates Secret "pulled-secret"               │
│  └────────────────────────────────┘                 │
└─────────────────────────────────────────────────────┘
```

Each managed cluster gets **its own** hub namespace (`eso-<cluster-name>`)
and its own ServiceAccount. No shared namespaces. The
`remoteNamespace` lock on the ClusterSecretStore prevents ESO from
ever reading secrets belonging to another cluster.

## Directory tree

```
external-secrets-store/
├── README.md
├── hub/
│   ├── base/
│   │   ├── kustomization.yaml
│   │   ├── serviceaccount.yaml
│   │   ├── role.yaml
│   │   └── rolebinding.yaml
│   └── overlays/
│       └── cluster-001a/
│           ├── kustomization.yaml
│           └── namespace.yaml
└── managed/
    ├── base/
    │   ├── kustomization.yaml
    │   ├── cluster-secret-store.yaml
    │   ├── token-secret.yaml
    │   └── external-secret-example.yaml
    └── overlays/
        └── cluster-001a/
            ├── kustomization.yaml
            └── patches.yaml
```

Add a new cluster by copying an overlay directory and changing the
cluster name in:
- `namespace.yaml` (hub) – the Namespace name
- `kustomization.yaml` (hub) – the `namespace:` field
- `patches.yaml` (managed) – the `remoteNamespace` value
- `kustomization.yaml` (managed) – the namespace value if it differs

## Requirements

- OpenShift 4.14+ (or any Kubernetes cluster with ESO installed)
- [Red Hat OpenShift External Secrets Operator][rh-eso] installed on
  each **managed** cluster (the operator is listed in the OpenShift
  OperatorHub). The upstream [external-secrets][eso] operator also
  works.
- Hub cluster accessible from managed clusters via the API server
  endpoint (typically port 6443).
- ACM is optional&mdash;you can apply the managed-cluster manifests
  directly. See [ACM / GitOps notes](#acm--gitops-notes) if you use
  ACM.

## Hub setup

### Step 1: Create a copy of the overlay for your cluster

```bash
cp -r ./external-secrets-store/hub/overlays/cluster-001a \
      ./external-secrets-store/hub/overlays/qat-staging-01
```

Edit `namespace.yaml` and change the Namespace name and description.
Edit `kustomization.yaml` and change the `namespace:` field to match.

### Step 2: Apply to the hub

```bash
# Preview
kubectl kustomize ./external-secrets-store/hub/overlays/cluster-001a

# Apply
oc apply -k ./external-secrets-store/hub/overlays/cluster-001a
```

Verify:

```bash
oc get sa -n eso-cluster-001a eso-reader
oc auth can-i -n eso-cluster-001a --as system:serviceaccount:eso-cluster-001a:eso-reader get secrets
# → yes
```

### Step 3: Push secrets

Create or copy secrets into the namespace:

```bash
oc -n eso-cluster-001a create secret generic my-secret \
  --from-literal=password=s3kr1t
```

### Step 4: Generate a ServiceAccount token

The managed cluster needs a bearer token to authenticate as
`eso-reader`. Generate one on the **hub**:

#### Token with expiry (recommended for rotation)

```bash
# Create a token request (no expiry = 1 year by default in OpenShift)
# Specify an expiry if you want shorter lived tokens.
oc create token eso-reader -n eso-cluster-001a --duration=8760h > ./token-cluster-001a
```

#### Long-lived static token

> **WARNING**: Static tokens do not expire automatically. You **must**
> rotate them manually. Use the expiry-dated approach above whenever
> possible.

```bash
# The SA's automatically created secret holds a static token.
SA_SECRET=$(oc get sa -n eso-cluster-001a eso-reader -o jsonpath='{.secrets[0].name}')
oc get secret -n eso-cluster-001a "$SA_SECRET" -o jsonpath='{.data.token}' | base64 -d > ./token-cluster-001a
```

Save the token to a file and keep it secure. You will inject it into
the managed cluster in the next section.

## Managed-cluster setup

### Step 1: Create a copy of the overlay for your cluster

```bash
cp -r ./external-secrets-store/managed/overlays/cluster-001a \
      ./external-secrets-store/managed/overlays/qat-staging-01
```

Edit `patches.yaml`:
- Set `server.url` to your hub cluster's API URL.
- Set `server.caBundle` to the base64-encoded CA certificate (see
  below).
- Set `remoteNamespace` to the hub namespace you created.

#### Getting the hub CA bundle

```bash
# Option A: from the router-ca (most portable)
oc config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' \
  > /tmp/hub-ca.p64

# Option B: from the API server serving CA
oc get secret -n openshift-kube-apiserver-operator node-kubeconfig \
  -o jsonpath='{.data.kubeconfig}' | base64 -d \
  | grep certificate-authority-data | awk '{print $2}' \
  > /tmp/hub-ca.p64
```

Paste the output into the `caBundle` field in `patches.yaml`.

### Step 2: Inject the token (without committing to Git)

Do **not** put the real token in `patches.yaml` or `token-secret.yaml`.
Apply the overlay first, then replace the token secret inline:

```bash
# Apply everything except the token (the placeholder is harmless)
oc apply -k ./external-secrets-store/managed/overlays/cluster-001a

# Then inject the real token
oc delete secret hub-token -n external-secrets
oc create secret generic hub-token -n external-secrets \
  --from-literal=token=$(cat ./token-cluster-001a)

# Verify the store reports ready
oc get clustersecretstore hub-store -o jsonpath='{.status.conditions[0]}'
```

Alternatively, use [Sealed Secrets][sealed-secrets],
[Mozilla SOPS][sops], or [Vault][vault] to encrypt the token file
before committing it.  Add the encrypted artifact to your repo and
decrypt it in your GitOps pipeline.

### Step 3: Create an ExternalSecret

The base includes an example ExternalSecret.  Adapt it to your needs:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-creds
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: hub-store
  target:
    name: db-creds
    creationPolicy: Owner
  data:
    - secretKey: username
      remoteRef:
        key: my-db-secret
        property: username
    - secretKey: password
      remoteRef:
        key: my-db-secret
        property: password
```

Apply it:

```bash
oc apply -f db-creds-external-secret.yaml
oc get secret db-creds
```

## Token rotation

### When to rotate

- At least every 90 days for compliance.
- Immediately if the token is compromised.
- When the ServiceAccount or Role changes on the hub.

### Rotate procedure

1. On the **hub**, generate a new token (see [Step 4 above](#step-4-generate-a-serviceaccount-token)).
2. On the **managed cluster**, update the secret:

```bash
oc delete secret hub-token -n external-secrets
oc create secret generic hub-token -n external-secrets \
  --from-literal=token=$(cat ./new-token)
```

3. Confirm the ClusterSecretStore transitions to `Ready`:

```bash
oc wait --for=condition=Ready clustersecretstore/hub-store --timeout=30s
```

## Apply commands summary

| Where | Command |
|---|---|
| Hub | `oc apply -k ./external-secrets-store/hub/overlays/cluster-001a` |
| Managed cluster | `oc apply -k ./external-secrets-store/managed/overlays/cluster-001a` |
| Hub (preview) | `kubectl kustomize ./external-secrets-store/hub/overlays/cluster-001a` |
| Managed (preview) | `kubectl kustomize ./external-secrets-store/managed/overlays/cluster-001a` |

## ACM / GitOps notes

If you manage the **managed-cluster** side with ACM:

1. Wrap the overlay output in a `Policy` / `ConfigurationPolicy`:

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: eso-hub-store
spec:
  remediationAction: enforce
  disabled: false
  policies:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: eso-hub-store
        spec:
          remediationAction: enforce
          severity: high
          namespaceSelector:
            exclude:
              - kube-*
              - openshift-*
          object-templates-raw: |
            {{- $secret := (fromClusterSecret "local-cluster" "external-secrets" "hub-token") }}
            ...
```

   However, injecting the bearer token through ACM policies requires
   the `fromClusterSecret` function or a Vault integration.  A simpler
   approach is to deploy the token with a separate HelmRelease or
    `Secret` manifest that ACM syncs from a secure Git branch.

2. Use a **PlacementRule** or **Placement** to target the policy to
   the correct cluster set.
3. Store encrypted tokens in Git (SealedSecret, SOPS) and decrypt them
   as part of the ACM policy pipeline on the hub.  The hub then pushes
   the decrypted token to the managed cluster.

Alternatively, skip ACM for secrets and manage the token out of band
via a short manual step after each rotation, while keeping everything
else (ClusterSecretStore, ExternalSecrets) in ACM policies.

## What this does not cover

- **PushSecret** capability.  This setup is read-only from the hub.
  Add PushSecret RBAC only if you explicitly need managed clusters to
  write back to the hub.
- **Multi-namespace stores.**  Each ClusterSecretStore locks to
  exactly one `remoteNamespace`.  A managed cluster that needs secrets
  from multiple hub namespaces gets one ClusterSecretStore per
  namespace.
- **Operator installation.**  Install the Red Hat OpenShift External
  Secrets Operator from OperatorHub on each managed cluster before
  applying this snippet.

[eso]: https://external-secrets.io
[rh-eso]: https://docs.openshift.com/container-platform/latest/external_security/external-secrets-operator.html
[sealed-secrets]: https://github.com/bitnami-labs/sealed-secrets
[sops]: https://github.com/mozilla/sops
[vault]: https://www.vaultproject.io