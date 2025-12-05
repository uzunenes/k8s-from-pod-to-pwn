# Kubernetes Battleground: k8s-from-pod-to-pwn

A hands-on series exploring how far you can go inside a Kubernetes cluster using "Living off the Land" techniques.

This repository contains the **lab scenarios** accompanying the blog/post series:

- Reproducible manifests (YAML) for each episode
- Commands to run from inside the pod
- Brief explanations and defense ("Fix") notes

## Content

We are currently focusing on:

1. **Ep1 – I've Landed in a Pod**: Env vars, ServiceAccount token, basic API discovery.
2. **Ep2 – What Am I Allowed To Do?** (Coming Soon): RBAC, SelfSubjectAccessReview, privilege escalation.

Future episodes will cover lateral movement, node access, cloud metadata abuse, and persistence.

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

**3. Install dependencies (inside VM):**

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

# Log out and log back in for group changes to take effect
exit
```

**4. Start the Lab (inside VM):**

Reconnect via SSH, then:

```bash
kind create cluster --name battleground

git clone https://github.com/uzunenes/k8s-from-pod-to-pwn.git
cd k8s-from-pod-to-pwn

kubectl apply -f episodes/ep1-landed-in-a-pod/manifests/
kubectl -n battleground exec -it deploy/demo-app -- sh
```

## Disclaimer

- All examples are for **educational and awareness purposes only**.
- Default settings in this lab are intentionally weakened to demonstrate risks. Do not use these configurations in production.