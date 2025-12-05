# Kubernetes Battleground: k8s-from-pod-to-pwn

A hands-on series exploring how far you can go inside a Kubernetes cluster using "Living off the Land" techniques.

This repository contains the **lab scenarios** accompanying the blog/post series:

- Reproducible manifests (YAML) for each episode
- Commands to run from inside the pod
- Brief explanations and defense ("Fix") notes

## Content

1. **[Ep1 – I've Landed in a Pod](episodes/ep1-landed-in-a-pod/README.md)**: Env vars, ServiceAccount token, basic API discovery.
2. **[Ep2 – The RBAC Mistake](episodes/ep2-rbac-misconfig/README.md)**: Exploiting overly permissive Roles to steal Secrets.
3. **[Ep3 – Breakout (Container Escape)](episodes/ep3-container-escape/README.md)**: Abusing `privileged` contexts and `hostPID` to escape to the host Node.
4. **[Ep4 – Node Domination](episodes/ep4-node-domination/README.md)**: Post-exploitation on the Node. Stealing Kubelet credentials, accessing host filesystem, and persistence via Static Pods.

## Prerequisites

- A Kubernetes environment:
  - [k3s](https://k3s.io/) - Tested and verified (v1.33.6+k3s1)
  - NetworkPolicy support included out of the box (kube-router)
- `kubectl` installed and connected to the cluster
- Basic understanding of Linux command line

> ℹ️ **Note:** This lab was fully tested on **k3s v1.33.6+k3s1** running on a GCP VM (Ubuntu 22.04). All attack and defense scenarios have been verified to work correctly.

This repo is designed for a **single user / single machine** scenario.

## Quickstart

1. Clone the repo:

   ```bash
   git clone https://github.com/uzunenes/k8s-from-pod-to-pwn.git
   cd k8s-from-pod-to-pwn
   ```

2. Install k3s (if not already installed):

   ```bash
   curl -sfL https://get.k3s.io | sh -
   
   # Configure kubectl
   mkdir -p ~/.kube
   sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
   sudo chown $(id -u):$(id -g) ~/.kube/config
   
   # Verify
   kubectl get nodes
   ```

3. Deploy the Episode 1 lab:

   ```bash
   kubectl apply -f episodes/ep1-landed-in-a-pod/manifests/
   ```

4. Enter the pod and follow the scenario:

   ```bash
   kubectl -n battleground exec -it deploy/demo-app -- sh
   ```

See `episodes/ep1-landed-in-a-pod/README.md` for detailed steps.

## Remote Lab Setup (GCP VM)

If you prefer a persistent remote environment, you can set up a VM on Google Cloud Platform.

**1. Create a VM:**

```bash
gcloud compute instances create k8s-from-pod-to-pwn-vm \
  --zone=europe-west1-b \
  --machine-type=e2-medium \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=40GB
```

**2. Connect via SSH:**

```bash
gcloud compute ssh ubuntu@k8s-from-pod-to-pwn-vm --zone=europe-west1-b
```

**3. Install k3s:**

```bash
# Install k3s
curl -sfL https://get.k3s.io | sh -

# Configure kubectl
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# Verify
kubectl get nodes
```

> ✅ **k3s v1.33+** includes kube-router which provides **NetworkPolicy support by default**.

> ⚠️ **Tested:** Kind's default CNI (kindnet) does **NOT** support NetworkPolicy. Defense scenarios using NetworkPolicy won't work on Kind unless you install Calico or Cilium.

**4. Start the Lab:**

```bash
git clone https://github.com/uzunenes/k8s-from-pod-to-pwn.git
cd k8s-from-pod-to-pwn

kubectl apply -f episodes/ep1-landed-in-a-pod/manifests/
kubectl -n battleground exec -it deploy/demo-app -- sh
```

## Disclaimer

- All examples are for **educational and awareness purposes only**.
- Default settings in this lab are intentionally weakened to demonstrate risks. Do not use these configurations in production.

## Optional: Policy Enforcement with Kyverno

Each episode includes a `defense/kyverno-policy.yaml` for cluster-wide policy enforcement.

> ⚠️ **Note:** Install Kyverno **after** testing the attack scenarios. Once policies are active, attack pods (privileged, hostPath, etc.) will be blocked at creation time.

```bash
# Install Kyverno
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.13.0/install.yaml

# Wait for pods
kubectl wait --for=condition=Ready pod -l app.kubernetes.io/instance=kyverno -n kyverno --timeout=120s

# Apply all policies
kubectl apply -f episodes/ep1-landed-in-a-pod/defense/kyverno-policy.yaml
kubectl apply -f episodes/ep2-rbac-misconfig/defense/kyverno-policy.yaml
kubectl apply -f episodes/ep3-container-escape/defense/kyverno-policy.yaml
kubectl apply -f episodes/ep4-node-domination/defense/kyverno-policy.yaml

# Verify
kubectl get clusterpolicy
```

**What gets blocked:**
- `restrict-service-account-token` – Blocks pods that mount SA tokens
- `restrict-secret-access` – Audits Roles granting Secret access
- `disallow-privileged-containers` – Blocks privileged + hostPID
- `disallow-host-path` – Blocks hostPath volume mounts
