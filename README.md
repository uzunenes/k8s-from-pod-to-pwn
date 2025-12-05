# Kubernetes Battleground: k8s-from-pod-to-pwn

A hands-on series exploring how far you can go inside a Kubernetes cluster using "Living off the Land" techniques.

This repository contains the **lab scenarios** accompanying the blog/post series:

- Reproducible manifests (YAML) for each episode
- Commands to run from inside the pod
- Brief explanations and defense ("Fix") notes

## Content

We are currently focusing on:

1. **[Ep1 – I've Landed in a Pod](episodes/ep1-landed-in-a-pod/README.md)**: Env vars, ServiceAccount token, basic API discovery.
2. **[Ep2 – The RBAC Mistake](episodes/ep2-rbac-misconfig/README.md)**: Exploiting overly permissive Roles to steal Secrets.
3. **[Ep3 – Breakout (Container Escape)](episodes/ep3-container-escape/README.md)**: Abusing `privileged` contexts, `hostPath` mounts, and `CAP_SYS_ADMIN` to escape to the host Node.

**Upcoming Advanced Scenarios:**

4. **Ep4 – Node Domination**: Post-exploitation on the Node. Stealing Kubelet credentials, accessing host filesystem, and persistence via Static Pods (since Cron/SSH are often missing).
5. **Ep5 – The HostNetwork Highway**: Abusing `hostNetwork: true` to bypass NetworkPolicies, sniff traffic, and access localhost services (Kubelet API).
6. **Ep6 – Cloud Lateral Movement**: Abusing Cloud Metadata (IMDS) to steal IAM credentials and pivot from Kubernetes to the Cloud Provider (AWS/GCP).
7. **Ep7 – Silent Persistence**: Backdooring Admission Controllers or mutating webhooks to inject sidecars silently.

## Prerequisites

- A Kubernetes environment (local setup recommended):
  - [kind](https://kind.sigs.k8s.io/)
  - or [minikube](https://minikube.sigs.k8s.io/docs/)
  - or [k3d](https://k3d.io/)
- `kubectl` installed and connected to the cluster
- Familiarity with basic Linux and bash commands

This repo is designed for a **single user / single machine** scenario.

## Quickstart (Local)

1. Clone the repo:

   ```bash
   git clone https://github.com/uzunenes/k8s-from-pod-to-pwn.git
   cd k8s-from-pod-to-pwn
   ```

2. Start a local cluster (e.g., using kind):

   ```bash
   kind create cluster --name battleground
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

**3. Install Kubernetes (choose one option):**

### Option A: k3s (Recommended - NetworkPolicy works out of the box)

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

### Option B: Kind (simpler, but no NetworkPolicy support)

```bash
# Update system & install tools
sudo apt update && sudo apt install -y docker.io kubectl git

# Enable Docker
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"

# Install kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Log out and log back in for group changes
exit
```

After reconnecting:

```bash
kind create cluster --name battleground
```

> ⚠️ **Note:** Kind's default CNI (kindnet) does **NOT** support NetworkPolicy. Defense scenarios using NetworkPolicy won't work unless you install Calico or Cilium.

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