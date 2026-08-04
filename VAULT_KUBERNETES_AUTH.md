# Vault Kubernetes authentication for the blog backend

This setup assumes Vault runs **outside** the Kubernetes cluster. Vault uses a
dedicated Kubernetes service account to call the TokenReview API. The backend,
Celery worker, and Celery beat authenticate as a separate workload identity.

| Identity | Namespace | Purpose |
| --- | --- | --- |
| `blog-backend` | `blog-dev` | Logs in to Vault with the `blog-backend-dev` role |
| `blog-backend` | `blog-stage` | Logs in to Vault with the `blog-backend-stage` role |
| `blog-backend` | `blog-prod` | Logs in to Vault with the `blog-backend-prod` role |
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
3. Check **Apply prerequisites to the shared Kubernetes cluster**.
4. Run the workflow and review its summary.

The `cluster-config` repository contains one `KUBECONFIG` repository secret for
the one physical cluster. The workflow is idempotent and does not read or print
the reviewer JWT. GitHub Environments remain deployment approval gates; they do
not contain duplicate copies of the kubeconfig.

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

## 3. Create least-privilege policies and roles

```sh
vault policy write blog-backend-dev - <<'EOF'
path "apps-kv/data/blog/dev/backend" {
  capabilities = ["read"]
}
EOF

vault write auth/kubernetes/role/blog-backend-dev \
  bound_service_account_names=blog-backend \
  bound_service_account_namespaces=blog-dev \
  policies=blog-backend-dev \
  ttl=1h
```

Repeat this pattern for `stage` and `prod`, changing all three of the policy
name, KV path, and bound namespace. Never bind one role to `blog-*` or give a
non-production policy access to the production KV path. Every role must bind
both the exact service account name and exact namespace.

Kubernetes authentication only gives a workload a Vault identity; it does not
place application values into the Pod. The recommended next step for this
application is Vault Secrets Operator: create one namespaced `VaultAuth` and
`VaultStaticSecret` per environment, synchronize to a namespaced Kubernetes
Secret, and reference that Secret from the backend, worker, and beat Pods. This
matches the application's existing environment-variable configuration. Enable
Kubernetes Secret encryption at rest and restrict Secret access with RBAC.

Vault Agent Injector is the alternative when secrets should be rendered to an
in-memory file instead of synchronized to Kubernetes Secrets. That option
requires the application entrypoints to load the rendered file.

## 4. Verify before deploying the application

Create a short-lived test token for the workload identity and exchange it for a
Vault token:

```sh
BLOG_JWT="$(kubectl create token blog-backend --namespace=blog-dev --duration=10m)"

vault write auth/kubernetes/login \
  role=blog-backend-dev \
  jwt="$BLOG_JWT"

unset BLOG_JWT
```

Then deploy the backend overlay, for example
`kubectl apply -k backend/overlays/dev`. The root prerequisite bundle must be
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

## Future non-production and production clusters

Bootstrap once per physical cluster, not once per application environment. The
non-production cluster contains `blog-dev` and `blog-stage`; the production
cluster contains only `blog-prod`. Each cluster has its own kubeconfig, reviewer
JWT, CA certificate, and Kubernetes API address.

If the existing external Vault serves both clusters, create one Kubernetes auth
mount per physical cluster:

```sh
vault auth enable -path=kubernetes-nonprod kubernetes
vault auth enable -path=kubernetes-prod kubernetes
```

Configure each mount with the reviewer JWT, CA certificate, and reachable API
address from its matching cluster. Bind the dev and stage roles under
`auth/kubernetes-nonprod`; bind the prod role only under
`auth/kubernetes-prod`. For example:

```sh
vault write auth/kubernetes-prod/config \
  kubernetes_host="$KUBERNETES_HOST" \
  kubernetes_ca_cert="$KUBERNETES_CA_CERT" \
  token_reviewer_jwt="$TOKEN_REVIEWER_JWT"

vault write auth/kubernetes-prod/role/blog-backend-prod \
  bound_service_account_names=blog-backend \
  bound_service_account_namespaces=blog-prod \
  policies=blog-backend-prod \
  ttl=1h
```

The application or secrets operator logs in through the mount matching its
physical cluster. Do not reconfigure one auth mount back and forth between
clusters because each mount stores only one Kubernetes API configuration.
