# Production Kubernetes Deployment Guide (Kubeadm + Containerd + Docker Registry)

This guide provides complete instructions for deploying the 3-Tier Web Application onto a production-ready **Kubeadm** Kubernetes cluster running the **containerd** container runtime and pulling images from a **Docker Registry** (Docker Hub, Harbor, AWS ECR, GitHub Container Registry, or a self-hosted private registry).

---

## 🏗️ Production Architecture Overview

```
                          [ External Clients ]
                                   │
                     ┌─────────────┴─────────────┐
                     │                           │
          (Method 1: Ingress-NGINX)      (Method 2: NodePort Proxy)
          07-ingress.yaml                08-proxy-nodeport.yaml (Port 30080)
                     │                           │
                     └─────────────┬─────────────┘
                                   │
                     ┌─────────────┴─────────────┐
                     │                           │
                     ▼ (/api, /health)           ▼ (/)
         [ backend-service:5000 ]    [ frontend-service:80 ]
         (Flask API - 2 Replicas)    (React + Nginx - 2 Replicas)
                     │
                     ▼ (Port 3306)
            [ mysql-service:3306 ]
            (MySQL 8.4 StatefulSet)
                     │
             [ mysql-pvc (5Gi) ]
         (Dynamic StorageClass / CSI)
```

---

## 📁 Manifest Directory Structure

| File | Resource | Description |
| :--- | :--- | :--- |
| [`00-namespace.yaml`](00-namespace.yaml) | `Namespace` | Creates isolated `three-tier` namespace |
| [`01-secret.yaml`](01-secret.yaml) | `Secret` | MySQL root & application credentials |
| [`02-configmap.yaml`](02-configmap.yaml) | `ConfigMap` | Backend DB settings & `init.sql` schema |
| [`03-mysql-pv-pvc.yaml`](03-mysql-pv-pvc.yaml) | `PVC` | 5Gi persistent volume using cluster StorageClass |
| [`04-mysql-statefulset.yaml`](04-mysql-statefulset.yaml) | `StatefulSet` + `Service` | MySQL 8.4 with `startupProbe`, `subPath: data`, and resource limits |
| [`05-backend-deployment.yaml`](05-backend-deployment.yaml) | `Deployment` + `Service` | Flask backend (2 replicas) with decoupled TCP liveness & HTTP readiness probes |
| [`06-frontend-deployment.yaml`](06-frontend-deployment.yaml) | `Deployment` + `Service` | React/Vite frontend (2 replicas) served via NGINX |
| [`07-ingress.yaml`](07-ingress.yaml) | `Ingress` | Standard `ingress-nginx` routing for `/api`, `/health`, `/` |
| [`08-proxy-nodeport.yaml`](08-proxy-nodeport.yaml) | `Deployment` + `Service` | Standalone NGINX reverse-proxy on NodePort `30080` with `/healthz` probe |
| [`09-network-policy.yaml`](09-network-policy.yaml) | `NetworkPolicy` | Zero-trust pod-to-pod microsegmentation |
| [`kustomization.yaml`](kustomization.yaml) | `Kustomization` | Unified deployment with image tag overrides |

---

## 📦 Step 1: Build & Push Images to Docker Registry

Because Kubeadm worker nodes pull images across the network via **containerd**, you must push your built images to a Docker registry accessible by your cluster nodes.

### 1. Define Your Registry & Build Images

Replace `your-registry` with your Docker Hub username, Harbor domain, or private registry URL (e.g. `registry.example.com/three-tier`):

```bash
# Example: Docker Hub
export REGISTRY="your-dockerhub-username"

# Example: Private Registry
# export REGISTRY="registry.example.com/three-tier"

# Build images
docker build -t $REGISTRY/three-tier-backend:1.0.0 ./backend
docker build -t $REGISTRY/three-tier-frontend:1.0.0 ./frontend
```

### 2. Log in and Push Images

```bash
docker login $REGISTRY
docker push $REGISTRY/three-tier-backend:1.0.0
docker push $REGISTRY/three-tier-frontend:1.0.0
```

---

## 🔑 Step 2: Configure Image Pull Secrets (For Private Registries)

If using a **private** registry that requires authentication:

1. Create a `docker-registry` secret in the `three-tier` namespace:

```bash
kubectl create namespace three-tier

kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=YOUR_USERNAME \
  --docker-password=YOUR_PASSWORD \
  --docker-email=YOUR_EMAIL \
  -n three-tier
```

2. Enable `imagePullSecrets` in `05-backend-deployment.yaml` and `06-frontend-deployment.yaml` under `spec.template.spec`:

```yaml
    spec:
      imagePullSecrets:
        - name: regcred
      containers:
        ...
```

---

## 🐳 Step 3: Containerd Runtime Configuration on Kubeadm Nodes

Kubeadm nodes manage containers via the `containerd` CRI plugin.

### Verify containerd on Worker Nodes
On each Kubeadm worker node, verify containerd is operating normally:

```bash
sudo crictl info
sudo crictl images
```

### Insecure / Self-Signed Private Registry (If Applicable)
If your private registry uses HTTP or an internal self-signed TLS certificate, configure `/etc/containerd/config.toml` on all worker nodes:

```toml
[plugins."io.containerd.grpc.v1.cri".registry.configs."registry.example.com".tls]
  insecure_skip_verify = true
```

Then restart containerd:
```bash
sudo systemctl restart containerd
```

---

## 💾 Step 4: Storage Setup in Kubeadm

In production Kubeadm clusters, persistent storage requires a CSI (Container Storage Interface) driver or StorageClass.

### 1. Verify Default StorageClass
Check if your cluster already has a default StorageClass:

```bash
kubectl get storageclass
```

You should see a StorageClass with `(default)`, for example:
- `local-path (default)` (Rancher Local Path Provisioner)
- `longhorn (default)` (Longhorn distributed storage)
- `nfs-client (default)` (NFS Subdir External Provisioner)

### 2. If No StorageClass Exists (Quick Install: Rancher Local Path Provisioner)
If your Kubeadm cluster does not have dynamic storage yet, install Rancher's lightweight single/multi-node local-path provisioner:

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.30/deploy/local-path-storage.yaml
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

The PVC defined in [`03-mysql-pv-pvc.yaml`](03-mysql-pv-pvc.yaml) will automatically bind dynamically to this StorageClass.

---

## 🛠️ Step 5: Configure Manifests & Kustomize

### 1. Update Registry Images in `kustomization.yaml`
Open [`k8s/kustomization.yaml`](kustomization.yaml) and uncomment/update the `images:` section with your registry:

```yaml
images:
  - name: three-tier-backend
    newName: your-registry/three-tier-backend
    newTag: 1.0.0
  - name: three-tier-frontend
    newName: your-registry/three-tier-frontend
    newTag: 1.0.0
```

### 2. Update Passwords in `01-secret.yaml`
Edit [`k8s/01-secret.yaml`](01-secret.yaml) with your strong production credentials:

```yaml
stringData:
  MYSQL_ROOT_PASSWORD: "YourStrongRootPassword"
  MYSQL_USER: "app_user"
  MYSQL_PASSWORD: "YourStrongAppPassword"
```

---

## 🚀 Step 6: Deploy to Kubernetes

Deploy all resources using Kustomize in a single command:

```bash
kubectl apply -k k8s/
```

### Manifest Details & Production Protections Built-In:
- **MySQL Reliability**:
  - `subPath: data` on `/var/lib/mysql` prevents collisions with `lost+found` directories.
  - `startupProbe` with `initialDelaySeconds: 20` and `failureThreshold: 30` allows up to 150 seconds for database creation and `init.sql` seeding without premature probe termination.
- **Backend High-Availability**:
  - 2 replicas with `RollingUpdate` strategy.
  - Decoupled `livenessProbe` (`tcpSocket: 5000`) avoids crash-loops during database restarts.
  - `readinessProbe` (`httpGet: /health`) guarantees user traffic is only routed when MySQL connectivity is confirmed.
- **Frontend High-Availability**:
  - 2 replicas with `RollingUpdate` strategy.
  - `livenessProbe` & `readinessProbe` checking NGINX root `/`.
- **Zero-Trust Network Isolation**:
  - `09-network-policy.yaml` restricts MySQL access exclusively to backend pods. Frontend pods cannot reach port 3306.

---

## 🌐 Step 7: Access the Application

### Option A: Using Ingress-NGINX (Recommended for Production)
If your Kubeadm cluster runs `ingress-nginx`:
1. Ensure `07-ingress.yaml` is applied (enabled by default).
2. Configure DNS for your cluster IP or NodePort.
3. Access:
   - `http://<Domain-or-IP>/` ➔ React Frontend
   - `http://<Domain-or-IP>/api/*` ➔ Flask Backend
   - `http://<Domain-or-IP>/health` ➔ Health Status

### Option B: Using Standalone NodePort Reverse Proxy
If your Kubeadm cluster does not run an Ingress Controller:
1. Ensure `08-proxy-nodeport.yaml` is active in `kustomization.yaml`.
2. Access the application directly on port **30080** on any worker node:
   ```text
   http://<Any-Worker-Node-IP>:30080
   ```

---

## 🔍 Verification, Operations & Troubleshooting

### 1. Verify Pods and PVC Status:
```bash
kubectl get pods,pvc,svc -n three-tier -o wide
```

Expected output:
```text
NAME                           READY   STATUS    RESTARTS   AGE
pod/backend-5b6d7bdbf-2srxm    1/1     Running   0          5m
pod/backend-5b6d7bdbf-mfbgr    1/1     Running   0          5m
pod/frontend-df4c58965-v8t76   1/1     Running   0          5m
pod/frontend-df4c58965-xfzwc   1/1     Running   0          5m
pod/mysql-0                    1/1     Running   0          5m
pod/proxy-7dcb4d5b7d-fvvsx     1/1     Running   0          5m
pod/proxy-7dcb4d5b7d-tf6bg     1/1     Running   0          5m

NAME                              STATUS   VOLUME
persistentvolumeclaim/mysql-pvc   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

### 2. Inspecting Containers with `crictl` on Kubeadm Nodes:
On any worker node running containerd:

```bash
# List pods in three-tier namespace:
sudo crictl pods --namespace three-tier

# List running containers:
sudo crictl ps --name backend
sudo crictl ps --name mysql

# View container logs directly from containerd:
sudo crictl logs <container-id>
```

### 3. Verify Database Connectivity & Health:
```bash
# Query backend /health endpoint:
kubectl exec -it -n three-tier deploy/backend -- python -c \
  "import urllib.request; print(urllib.request.urlopen('http://localhost:5000/health').read().decode())"
```

Expected response:
```json
{"database":"connected","service":"flask-backend","status":"ok"}
```

### 4. Verify Database Schema Seeding:
```bash
kubectl exec -it -n three-tier mysql-0 -- mysql -uapp_user -pchange_app_secure_password -D registration_db -e "SHOW TABLES;"
```

Expected response:
```text
+---------------------------+
| Tables_in_registration_db |
+---------------------------+
| users                     |
+---------------------------+
```



######
Docker Registry Workflow:

Building and tagging images for Docker Hub, Harbor, AWS ECR, or self-hosted registries:
bash
docker build -t your-registry/three-tier-backend:1.0.0 ./backend
docker build -t your-registry/three-tier-frontend:1.0.0 ./frontend
docker push your-registry/three-tier-backend:1.0.0
docker push your-registry/three-tier-frontend:1.0.0
Configuring imagePullSecrets (regcred) for private registries.
Using 

k8s/kustomization.yaml
 to customize image repositories without altering individual YAML manifests.
Containerd Runtime Instructions for Kubeadm Nodes:

Inspecting node containers and pods directly via crictl (crictl info, crictl pods --namespace three-tier, crictl ps).
Configuring /etc/containerd/config.toml for insecure/internal registries if using private HTTP or self-signed TLS endpoints.
Storage Setup:

Instructions for dynamic CSI storage (e.g. Rancher local-path-provisioner, Longhorn, or NFS) that automatically binds to 

03-mysql-pv-pvc.yaml
.
Production Stability Fixes Included in the Manifests:

MySQL 8.4: Added startupProbe (up to 150 seconds) and subPath: data on /var/lib/mysql to avoid probe kills during initialization and prevent collisions with filesystem metadata.
Backend API: Decoupled livenessProbe (tcpSocket: 5000) from database state to prevent restart loops, while preserving readinessProbe (httpGet: /health) so user traffic is only routed when MySQL is connected.
Standalone Proxy: Added internal /healthz endpoint to 

08-proxy-nodeport.yaml
 to allow reliable testing on NodePort 30080 without an ingress controller.