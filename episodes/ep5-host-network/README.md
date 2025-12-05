# Episode 5 – The HostNetwork Highway

In previous episodes, we dealt with file system isolation. Now, let's break **Network Isolation**.

In this episode, we assume:
> You have compromised a pod (or can deploy one) that has `hostNetwork: true`.
> This is often used by monitoring agents, CNI plugins, or load balancers to improve performance.

Our goal here is to:
- **Bypass Network Namespaces**: See the Node's network interfaces directly.
- **Access Localhost Services**: Reach services bound to `127.0.0.1` on the Node (which are usually not accessible to pods).
- **Bypass NetworkPolicies**: Since we share the Node's IP, we often bypass pod-level firewall rules.

## 1. Setup the Lab

We deploy a simple Ubuntu pod with `hostNetwork: true`.

From the root of this repo:

```bash
kubectl apply -f episodes/ep5-host-network/manifests/
```

## 2. Enter the Pod

```bash
kubectl -n battleground exec -it deploy/net-spy -- bash
```

## 3. The Attack: Network Reconnaissance

### 3.1. "Where am I?" (Network View)

Check the network interfaces:

```bash
apt-get update && apt-get install -y iproute2 curl
ip addr
```

**Observation:**
- In a normal pod, you see `eth0` and `lo`.
- Here, you will see **ALL** interfaces of the Node (`eth0`, `docker0`, `veth...`, `cni0`).
- You have the **Node's IP address**.

### 3.2. Accessing Localhost Services

Many services bind to `127.0.0.1` for security, assuming "only processes on this machine can reach me".
Guess what? **You are on this machine.**

Try to access the Kubelet's healthz endpoint (usually port 10248):

```bash
curl -v http://127.0.0.1:10248/healthz
```

**Output:** `OK`

Try to access the Kubelet's ReadOnly API (port 10255 - often disabled in modern clusters, but worth a try):

```bash
curl -v http://127.0.0.1:10255/pods
```

If successful, this dumps **ALL** pods running on the node, including their environment variables (which might contain secrets!).

### 3.3. Sniffing Traffic (The "Wiretap")

Since you are on the host network, if you have `CAP_NET_RAW` or `privileged` (often combined with hostNetwork), you can sniff traffic.

```bash
# If tcpdump is installed (or you install it)
tcpdump -i any port 80
```

You would see traffic from **other pods** on this node.

## 4. Fix

- **Restrict hostNetwork:** Use **Pod Security Standards (PSS)** to disallow `hostNetwork: true` in the "Baseline" or "Restricted" profiles.
- **Gatekeeper/Kyverno:** Explicitly block `hostNetwork: true` unless whitelisted for specific system components.
- **Bind Interfaces:** Ensure system services (like Kubelet) bind to specific interfaces/ports and require authentication, even on localhost.
