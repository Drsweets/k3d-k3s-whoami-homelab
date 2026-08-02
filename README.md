Install dependencies inside Ubuntu VM

# Update system
sudo apt update && sudo apt upgrade -y

# Install docker
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Install k3d
wget -q https://github.com/k3d-io/k3d/releases/latest/download/k3d-linux-amd64
chmod +x k3d-linux-amd64
sudo mv k3d-linux-amd64 /usr/local/bin/k3d

k3d cluster create -c k3d-config.yaml
# Verify cluster
k3d cluster list
kubectl get nodes

Install ArgoCD (GitOps Core Demo)

# Create namespace
kubectl create ns argocd

# Install ArgoCD stable manifests
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods ready
kubectl get pods -n argocd --watch

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# Expose ArgoCD UI via NodePort for homelab access
kubectl patch svc argocd-server -n argocd -p '{"spec":{"type":"NodePort","ports":[{"port":443,"targetPort":8080,"nodePort":30443}]}}'

Open UI: https://<VM_IP>:30443
Login user: admin

Create ArgoCD Application CRD
kubectl apply -f argocd-app.yaml

Install Prometheus & Grafana via Helm

# Install helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Add prometheus helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create namespace
kubectl create ns monitoring

# Install kube-prometheus stack
helm install kube-prometheus prometheus-community/kube-prometheus-stack \
-n monitoring \
-f kube-prometheus-values.yaml

Expose Grafana NodePort 
kubectl patch svc kube-prometheus-grafana -n monitoring -p '{"spec":{"type":"NodePort","ports":[{"port":80,"targetPort":3000,"nodePort":30030}]}}'

Grafana access: http://<VM_IP>:30030
User: admin, password: Homelab@123
Import official Kubernetes cluster dashboard (ID: 7249)


Test full stack
Visit whoami ingress http://<VM_IP>:8080
Check ArgoCD UI: application status Synced & Healthy
Open Grafana and view node/pod resource metrics
Make git change to whoami deployment → watch auto sync

