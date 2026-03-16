# Kubernetes Practice Demo

A minimal repo to run a real Kubernetes cluster locally with **kind** (with multiple nodes so you can see control-plane vs workers), deploy an app, expose it with a Service, and scale it live.

## Prerequisites

You need **Docker**, **kubectl**, and **kind**. Install both for your platform below.

### Installing kubectl and kind

#### macOS (Unix)

```bash
# Homebrew (install from https://brew.sh if needed)
# Docker should already be installed
brew install kubectl kind
```

#### Linux

**docker:**
```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**kubectl:**

```bash
# Download latest stable (check https://kubernetes.io/docs/tasks/tools/ for your arch)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

**kind:**

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

(Get the latest kind version from [releases](https://github.com/kubernetes-sigs/kind/releases).)

#### WSL (Windows Subsystem for Linux)

Use the **Linux** instructions above inside your WSL terminal (Ubuntu/Debian or other). The same `curl` commands work; ensure you use the `linux-amd64` (or `linux-arm64` on ARM) binaries.

Optional — install via package manager in WSL (e.g. Ubuntu):

```bash
# kubectl
sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update && sudo apt-get install -y kubectl

# kind — use the Linux curl method above (no apt package)
```

Verify:

```bash
kubectl version --client
kind --version
```

## 1. Start a real Kubernetes cluster locally

Create a **multi-node** cluster (1 control-plane node + 2 worker nodes) so you can see different node roles:

**macOS**:

```bash
kind create cluster --name k8s-practice --config kind-config.yaml
```

**linux/wsl**:

```bash
sudo kind create cluster --name k8s-practice --config kind-config.yaml
mkdir -p ~/.kube
sudo kind export kubeconfig --name k8s-practice --kubeconfig ~/.kube/config
sudo chown 1000:1000 ~/.kube/config
```

Verify:

```bash
kubectl cluster-info
kubectl get nodes
```

You should see three nodes: one **control-plane** and two **worker** nodes.

### Control plane vs worker nodes

| | **Control plane** | **Worker** |
|---|-------------------|------------|
| **Role** | Runs the cluster’s “brain”: API server, scheduler, controller-manager, etcd. | Runs your workloads (Pods). |
| **Responsibilities** | Accepts requests (e.g. from `kubectl`), decides where to run pods, keeps desired state. | Runs kubelet + container runtime; starts and stops containers. |
| **User workloads** | Typically **no** — system components only (in production you often taint control-plane so pods don’t schedule there). | **Yes** — your Deployments and Pods run here. |

- **Control plane** = manage the cluster. **Worker** = run the apps.
- In this demo, kind may schedule some pods on the control-plane node too (single cluster, no taints). In production you’d usually have multiple workers and keep workloads off the control plane.

See which node each pod is on:

```bash
kubectl get pods -l app=demo-app -o wide
```

The `NODE` column shows the node name; you’ll see worker (and possibly control-plane) names once the app is deployed.

## 2. Apply the Deployment

```bash
kubectl apply -f manifests/deployment.yaml
```

Check that pods are running:

```bash
kubectl get pods -l app=demo-app
kubectl get deployment demo-app
```

## 3. Create the Service

```bash
kubectl apply -f manifests/service.yaml
```

Check the service:

```bash
kubectl get svc demo-app
```

## 4. Access it in the browser

Port-forward the service to your machine:

```bash
kubectl port-forward svc/demo-app 8080:80
```

Open **http://localhost:8080** in your browser. You should see: **Hello from Kubernetes!**

(Leave the port-forward running in that terminal, or run it in the background.)

## 5. Change replicas live

Scale up:

```bash
kubectl scale deployment demo-app --replicas=5
```

Watch pods come up:

```bash
kubectl get pods -l app=demo-app -w
```

Scale down:

```bash
kubectl scale deployment demo-app --replicas=1
```

Refresh the browser and hit the URL a few times — you’ll get the same response from different pods (round-robin via the Service).

## Cleanup

Delete the app resources, then delete the kind cluster (use the **same name** you used in step 1):

**macOS:**

```bash
kubectl delete -f manifests/
kind delete cluster --name k8s-practice
```

**linux/wsl:**

```bash
kubectl delete -f manifests/
sudo kind delete cluster --name k8s-practice
rm -rf ~/.kube
```

To see your cluster name if you’re not sure: `kind get clusters`. Use that exact name in the delete command.
