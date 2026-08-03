# Vault Kubernetes authentication for the blog backend

This setup assumes Vault runs **outside** the Kubernetes cluster. Vault uses a
dedicated Kubernetes service account to call the TokenReview API. The backend,
Celery worker, and Celery beat authenticate as a separate workload identity.

| Identity | Namespace | Purpose |
| --- | --- | --- |
| `blog-backend` | `blog` | Logs in to Vault and receives the `blog-backend` policy |
| `vault-token-reviewer` | `vault-auth` | Used only by Vault to call Kubernetes TokenReview |

## 1. Apply Kubernetes prerequisites

Apply these resources before configuring `auth/kubernetes` in Vault and before
deploying the backend:

```sh
kubectl apply -k .
kubectl wait --for=jsonpath='{.data.token}' secret/vault-token-reviewer \
  --namespace=vault-auth --timeout=60s
```

The committed Secret manifest contains no credential. Kubernetes fills its
`token`, `ca.crt`, and `namespace` fields after it is applied.

For a remote cluster, the same step can be run manually from GitHub Actions:

1. Open **Actions** and select **Initial Cluster Bootstrap**.
2. Select **Run workflow**.
3. Select the target GitHub environment: `dev`, `staging`, or `prod`.
4. Check **Apply the cluster prerequisite manifests**.
5. Run the workflow and review its summary.

Each GitHub environment must contain a `KUBECONFIG` secret for its own cluster.
The workflow loads secrets from the selected environment, is idempotent, and
does not read or print the reviewer JWT. Configure protection and approval rules
independently on the `dev`, `staging`, and `prod` GitHub environments.

## 2. Configure Vault without writing the reviewer JWT to disk

Run this from a trusted administrative shell with `VAULT_ADDR` set and with a
Vault token that can configure auth methods. The shell variables exist only for
this process; do not print them, enable shell tracing, or save them in history.

```sh
TOKEN_REVIEWER_JWT="$(kubectl get secret vault-token-reviewer \
  --namespace=vault-auth \
  --output='go-template={{ index .data "token" | base64decode }}')"

KUBERNETES_CA_CERT="$(kubectl get secret vault-token-reviewer \
  --namespace=vault-auth \
  --output='go-template={{ index .data "ca.crt" | base64decode }}')"

KUBERNETES_HOST="$(kubectl config view --raw --minify \
  --output='jsonpath={.clusters[0].cluster.server}')"

# Run this once. Skip it if auth/kubernetes is already enabled.
vault auth enable kubernetes

vault write auth/kubernetes/config \
  kubernetes_host="$KUBERNETES_HOST" \
  kubernetes_ca_cert="$KUBERNETES_CA_CERT" \
  token_reviewer_jwt="$TOKEN_REVIEWER_JWT"

unset TOKEN_REVIEWER_JWT KUBERNETES_CA_CERT KUBERNETES_HOST
```

Vault stores the reviewer JWT inside its encrypted storage. Do not store a copy
in Git, a GitHub Actions secret, a ConfigMap, an application Secret, or a local
file. Access to the Kubernetes Secret should be restricted to cluster admins.

## 3. Create the least-privilege policy and role

```sh
vault policy write blog-backend - <<'EOF'
path "apps-kv/data/blog/backend" {
  capabilities = ["read"]
}
EOF

vault write auth/kubernetes/role/blog-backend \
  bound_service_account_names=blog-backend \
  bound_service_account_namespaces=blog \
  policies=blog-backend \
  ttl=1h
```

If the existing secret lives at another KV path, change only the policy path.
The role must continue to bind both the exact service account and namespace.

## 4. Verify before deploying the application

Create a short-lived test token for the workload identity and exchange it for a
Vault token:

```sh
BLOG_JWT="$(kubectl create token blog-backend --namespace=blog --duration=10m)"

vault write auth/kubernetes/login \
  role=blog-backend \
  jwt="$BLOG_JWT"

unset BLOG_JWT
```

Then deploy the backend overlay, for example
`kubectl apply -k backend/overlays/prod`. The root prerequisite bundle must be
applied first because Kustomize bases cannot import files above their directory.
The backend pods receive rotating, projected `blog-backend` tokens
automatically.

## Rotation and recovery

The manually-created `kubernetes.io/service-account-token` Secret is long-lived.
Rotate it if it may have been exposed, when the cluster/service account is
re-created, and as part of a scheduled credential-rotation procedure:

1. Create a replacement token Secret for the same reviewer service account.
2. Write the replacement JWT and CA certificate to `auth/kubernetes/config`.
3. Verify a `blog-backend` login succeeds.
4. Delete the old token Secret.

Do not delete or replace the active reviewer Secret before Vault has been
updated, or all Kubernetes logins through this auth mount will fail.

## If Vault runs inside this Kubernetes cluster

Do not use the long-lived reviewer Secret. Bind `system:auth-delegator` to the
service account used by the Vault server pods, omit `token_reviewer_jwt` when
configuring the auth method, and let Vault read its rotating local pod token.
In that topology, the reviewer service account and Secret in this repository
should be removed.

## Multiple independent clusters

Run **Initial Cluster Bootstrap** once for each GitHub environment. Because the
clusters are independent, using the same Kubernetes names (`blog`, `vault-auth`,
`blog-backend`, and `vault-token-reviewer`) does not create a conflict. Each
cluster produces its own reviewer JWT and CA certificate.

If each cluster also has its own independent Vault deployment, every Vault can
use the standard `auth/kubernetes` mount shown above.

If one shared Vault serves all three Kubernetes clusters, create a separate
Kubernetes auth mount for each API server:

```sh
vault auth enable -path=kubernetes-dev kubernetes
vault auth enable -path=kubernetes-staging kubernetes
vault auth enable -path=kubernetes-prod kubernetes
```

Configure each mount using the reviewer JWT, CA certificate, and reachable API
address from its matching cluster. For example, production uses:

```sh
vault write auth/kubernetes-prod/config \
  kubernetes_host="$KUBERNETES_HOST" \
  kubernetes_ca_cert="$KUBERNETES_CA_CERT" \
  token_reviewer_jwt="$TOKEN_REVIEWER_JWT"

vault write auth/kubernetes-prod/role/blog-backend \
  bound_service_account_names=blog-backend \
  bound_service_account_namespaces=blog \
  policies=blog-backend-prod \
  ttl=1h
```

The application must log in through the mount matching its cluster, such as
`auth/kubernetes-prod/login`. A single Kubernetes auth mount must not be
reconfigured back and forth between clusters because each mount has only one
Kubernetes API configuration.
