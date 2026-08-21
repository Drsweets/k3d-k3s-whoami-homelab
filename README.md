# k3d-k3s-whoami-homelab

A lightweight, multi-node Kubernetes homelab built with **K3d (K3s running inside Docker)**, deployed on an Ubuntu VM hosted in Proxmox.

This repo follows a **stable, manual workflow**: provision the cluster, then `kubectl apply` a simple `whoami` demo workload. No GitOps, no monitoring stack — just a reliable, repeatable setup for lab work and quick demos (e.g. LinkedIn posts).

> **Target environment:** Proxmox Ubuntu 24.04 VM with Docker pre-installed.

---

## Table of Contents

- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Step 1 — Clone the Repository](#step-1--clone-the-repository)
- [Step 2 — Fix the TLS-SAN IP](#step-2--fix-the-tls-san-ip-in-k3d-config)
- [Step 3 — Create the Cluster](#step-3--create-the-k3d-multi-node-cluster)
- [Step 4 — Deploy the whoami App](#step-4--deploy-the-whoami-application)
- [Step 5 — Test the Application](#step-5--test-the-application)
- [Step 6 — Debug Commands](#step-6--debug-commands)
- [Step 7 — Teardown](#step-7--teardown)

---

## Repository Structure

```
k3d-k3s-whoami-homelab/
├── README.md                 # This file
├── 00-k3d-config.yaml        # k3d cluster definition (1 server + 1 agent)
├── 01-namespace.yaml         # Creates the homelab-demo namespace
├── 02-whoami-deployment.yaml # Reference only: Deployment (replicas: 2, whoami-svc)
├── 03-whoami-service.yaml    # Reference only: Service (name: whoami-svc)
├── 04-ingress.yaml           # Reference only: Ingress (host: homelab.local)
└── 05-whoami.yaml            # RECOMMENDED: combined Deployment + Service + Ingress
```

**Which manifests should I apply?**
Use `05-whoami.yaml`. It's a single, self-contained file with consistent naming (Service `whoami`, host `whoami.local`, namespace `homelab-demo`).

The split files `02`–`04` use different naming conventions (`whoami-svc`, `homelab.local`) and exist purely as reference examples. **Do not apply both sets** — they'll create conflicting resources in the cluster.

---

## Prerequisites

Docker is assumed to already be installed on the VM. Install the remaining tooling:

```bash
sudo apt update
sudo apt install -y git curl

# Install k3d
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Add your user to the docker group (avoids sudo for docker/k3d)
sudo usermod -aG docker $USER
newgrp docker

# Validate installs
docker --version
k3d version
kubectl version --client
git --version
```

---

## Step 1 — Clone the Repository

```bash
git clone https://github.com/Drsweets/k3d-k3s-whoami-homelab.git
cd k3d-k3s-whoami-homelab
ls -la
```

---

## Step 2 — Fix the TLS-SAN IP in k3d Config

`00-k3d-config.yaml` ships with a hard-coded, foreign LAN IP (`--tls-san=192.168.1.13`). Left unchanged, this causes an `x509: certificate is valid for...` error when `kubectl` talks to the K3s API server from a different network.

**1. Get your VM's private IP:**

```bash
hostname -I
```

**2. Edit the config:**

```bash
nano 00-k3d-config.yaml
```

**3. Replace the IP after `--tls-san=`** with your VM's actual LAN IP. The corrected file should look like this:

```yaml
apiVersion: k3d.io/v1alpha5
kind: Simple
metadata:
  name: k3s-homelab
servers: 1
agents: 1
image: rancher/k3s:v1.30.2-k3s1
ports:
  - port: 80:80
    nodeFilters:
      - loadbalancer
  - port: 443:443
    nodeFilters:
      - loadbalancer
options:
  k3s:
    extraArgs:
      - arg: --tls-san=<YOUR_VM_PRIVATE_IP>
        nodeFilters:
          - server:*
```

> Replace `<YOUR_VM_PRIVATE_IP>` with the output of `hostname -I`.
>
> **Port conflict?** If host ports 80/443 are already in use on the VM, remap them — e.g. `8080:80` and `8443:443`.

---

## Step 3 — Create the k3d Multi-Node Cluster

```bash
k3d cluster create -c 00-k3d-config.yaml

# Confirm the cluster exists
k3d cluster list

# Write and export kubeconfig for kubectl
export KUBECONFIG=$(k3d kubeconfig write k3s-homelab)

# Validate cluster health
kubectl get nodes
kubectl get pods -A
```

**Expected result:** one server node and one agent node, both in `Ready` state.

### Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `x509: certificate signed by unknown authority` | Wrong `--tls-san` IP | Destroy the cluster, fix the IP in the yaml, recreate |
| `bind: address already in use` on port 80/443 | Host port conflict | Remap host ports in `00-k3d-config.yaml` (e.g. `8080:80`) |

Destroy and recreate the cluster:

```bash
k3d cluster delete k3s-homelab
k3d cluster create -c 00-k3d-config.yaml
```

---

## Step 4 — Deploy the whoami Application

All resources are hard-coded to the `homelab-demo` namespace. Create the namespace first, then apply the combined manifest:

```bash
kubectl apply -f 01-namespace.yaml
kubectl apply -f 05-whoami.yaml
```

Verify everything came up cleanly:

```bash
kubectl get pods -n homelab-demo
kubectl get svc -n homelab-demo
kubectl get ingress -n homelab-demo
kubectl get endpoints -n homelab-demo
```

> `kubectl get endpoints` is the key check — it confirms the Service is actually selecting and routing to the backend whoami pods.

**Optional — set the namespace as your default context** so you don't need `-n homelab-demo` on every command:

```bash
kubectl config set-context --current --namespace=homelab-demo
```

---

## Step 5 — Test the Application

### From inside the VM

```bash
curl -H "Host: whoami.local" http://127.0.0.1
```

Or add a hosts entry and hit the hostname directly:

```bash
echo "127.0.0.1 whoami.local" | sudo tee -a /etc/hosts
curl http://whoami.local
```

### From an external laptop / workstation

Add this line to your local machine's hosts file (replace with your VM's actual IP):

```
<VM_PRIVATE_IP>  whoami.local
```

Then open `http://whoami.local` in a browser.

> If the Ingress shows `<none>` under `ADDRESS`, that's expected for k3d — Traefik binds via the loadbalancer container. If you get a 404 instead of the whoami response, check the Traefik logs:
> ```bash
> kubectl logs -n kube-system -l app=traefik
> ```

---

## Step 6 — Debug Commands

```bash
# View application pod logs
kubectl logs -n homelab-demo -l app=whoami

# Inspect Ingress events and errors
kubectl describe ingress -n homelab-demo whoami-ingress

# List all resources in the namespace
kubectl get all -n homelab-demo

# View the underlying k3d Docker containers
docker ps

# Exec into the K3s server node container
docker exec -it $(docker ps | grep k3d-k3s-homelab-server-0 | awk '{print $1}') sh
```

---

## Step 7 — Teardown

### Option A — Remove only the whoami workload (keep the cluster running)

```bash
kubectl delete -f 05-whoami.yaml
kubectl delete -f 01-namespace.yaml
```

### Option B — Destroy the entire k3d cluster (full reset)

```bash
k3d cluster delete k3s-homelab
docker system prune -f
```

This removes all Kubernetes resources, every k3d Docker container (server, agent, loadbalancer), and the kubeconfig entry.

### Option C — Remove local project files

```bash
cd ~
rm -rf ./k3d-k3s-whoami-homelab
```

### Full teardown, one-liner

```bash
kubectl delete namespace homelab-demo && k3d cluster delete k3s-homelab && rm -rf ~/k3d-k3s-whoami-homelab
```

---
