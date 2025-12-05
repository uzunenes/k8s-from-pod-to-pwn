# Episode 1 – I've Landed in a Pod

In this episode, we assume:

> You have successfully gained shell access to a pod.
>
> (Whether via application vulnerability, supply chain attack, or misconfiguration – **how** you got here is covered in Ep0.)

Our goal here is to:

- Use only **standard Linux tools** (env, cat, curl, mount, etc.)
- Discover **where we are** and **which ServiceAccount we are running as**
- See **how much of the Kubernetes API we can access**

## 1. Setup the Lab

From the root of this repo:

```bash
kubectl apply -f episodes/ep1-landed-in-a-pod/manifests/
```

This command:

- Creates a namespace named `battleground`
- Deploys a simple `busybox`-based Deployment named `demo-app`

Check the pod status:

```bash
kubectl -n battleground get pods
```

Once it is `Running`, we can open a shell inside.

## 2. Enter the Pod

```bash
kubectl -n battleground exec -it deploy/demo-app -- sh
```

From now on, all commands are run **inside the pod**.

## 3. Dev View: What Can I See?

### 3.1. Environment Variables (env)

```sh
env | sort
```

Look for:

- `KUBERNETES_SERVICE_HOST` and `KUBERNETES_SERVICE_PORT`
- Application-specific env vars (DB hosts, credentials, etc. – not present in this lab, but critical in real life)

### 3.2. DNS and Resolver

```sh
cat /etc/resolv.conf
```

This reveals:

- The Cluster DNS service IP
- Search domains (e.g., `svc.cluster.local`)

### 3.3. ServiceAccount Token

By default, Kubernetes mounts a ServiceAccount token volume to pods (unless `automountServiceAccountToken: false` is set).

```sh
ls -R /var/run/secrets/kubernetes.io/serviceaccount

cat /var/run/secrets/kubernetes.io/serviceaccount/namespace
cat /var/run/secrets/kubernetes.io/serviceaccount/token | head -c 40; echo "..."
```

What you see:

- Which namespace you are in (`namespace` file)
- Which ServiceAccount you are running as (default in this lab)
- The Bearer Token you can use to authenticate against the API

## 4. First Contact with Kubernetes API (curl)

First, let's export the token and CA certificate to variables:

```sh
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
APISERVER="https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}"

echo $APISERVER
```

### 4.1. See the API Root

```sh
curl --cacert "$CACERT" \
  -H "Authorization: Bearer $TOKEN" \
  "$APISERVER/api"
```

You should see the API versions and paths.

**Example Output:**

```json
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "172.18.0.2:6443"
    }
  ]
}
```

### 4.2. What's in My Namespace?

```sh
NAMESPACE=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)

echo "Namespace: $NAMESPACE"

curl --cacert "$CACERT" \
  -H "Authorization: Bearer $TOKEN" \
  "$APISERVER/api/v1/namespaces/$NAMESPACE/pods"
```

If this call is successful:

- You can list pods in your own namespace
- You know you are "not alone"

**However, in a secure default environment (like this lab), you will likely see:**

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "pods is forbidden: User \"system:serviceaccount:battleground:default\" cannot list resource \"pods\" in API group \"\" in the namespace \"battleground\"",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
}
```

This confirms that the default ServiceAccount has no permissions to list pods.

## 5. Attack: Where Do I Go From Here?

What we've shown so far might look like **normal application behavior**:

- Reading env vars
- Accessing the ServiceAccount token
- Fetching resources from the API

However, in practice:

- An attacker who lands in a pod via an app vulnerability
- Starts cluster reconnaissance with this information
- If RBAC is misconfigured, they can go much further (covered in Ep2)

Key takeaway:

> Just because you "only landed in a pod" doesn't mean you can "only affect this application".

## 6. Fix: What Can Platform / Ops Teams Do?

Some preventive measures at this stage:

- **Do not grant permissions to the default ServiceAccount**
- Define a specific ServiceAccount for the application and grant only necessary permissions
- If not needed:
  - Set `automountServiceAccountToken: false` in the Pod spec
- Manage sensitive configs (credentials, endpoints) via:
  - Secrets + RBAC,
  - Or external secret management (Vault, KMS, etc.) instead of env vars

In Ep2, we will explore **what this ServiceAccount is allowed to do** and how to escalate privileges from here.

## 7. Blue Team Corner (Defense in Depth) 🛡️

We have added a `defense/` folder with secure configurations to prevent these issues.

1.  **Secure Deployment (`defense/secure-deployment.yaml`):**
    - Disables ServiceAccount token mounting (`automountServiceAccountToken: false`).
    - Runs as non-root user (`runAsNonRoot: true`).
    - Drops all Linux capabilities.
    - Uses read-only root filesystem.

2.  **Network Policy (`defense/network-policy.yaml`):**
    - Denies all ingress traffic (since this is an internal app).
    - Allows egress only to DNS (UDP 53), blocking access to the API Server or external C2 servers.

3.  **Policy as Code (`defense/kyverno-policy.yaml`):**
    - A Kyverno policy that enforces `automountServiceAccountToken: false` for all pods.

**Try it out:**
```bash
kubectl apply -f episodes/ep1-landed-in-a-pod/defense/
```

### 7.1. Verifying the Defense

After applying the secure deployment, verify the protections:

**Test 1: ServiceAccount Token Not Mounted**
```bash
kubectl -n battleground exec deploy/demo-app-secure -- ls /var/run/secrets/kubernetes.io/serviceaccount/
```

**Expected Output:**
```
ls: cannot access '/var/run/secrets/kubernetes.io/serviceaccount/': No such file or directory
```

**Test 2: Running as Non-Root User**
```bash
kubectl -n battleground exec deploy/demo-app-secure -- id
```

**Expected Output:**
```
uid=10001 gid=0(root) groups=0(root)
```

**Test 3: Read-Only Filesystem**
```bash
kubectl -n battleground exec deploy/demo-app-secure -- touch /test-file
```

**Expected Output:**
```
touch: cannot touch '/test-file': Read-only file system
```

**Test 4: API Access Without Token**
```bash
kubectl -n battleground exec deploy/demo-app-secure -- \
  sh -c 'curl -s -k https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT/api'
```

**Expected Output:**
```json
{
  "kind": "Status",
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/api\"",
  "reason": "Forbidden",
  "code": 403
}
```

> ⚠️ **Important Note about NetworkPolicy:**
> The NetworkPolicy will only work if your CNI plugin supports it. **Kind's default CNI (kindnet) does NOT support NetworkPolicy**. For production or proper testing, use Calico, Cilium, or Weave Net.
>
> To install Calico on Kind:
> ```bash
> kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
> ```

