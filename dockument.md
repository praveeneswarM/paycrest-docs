# PayCrest LMS — Project Implementation & Operations Manual

> **Version:** 1.0 | **Audience:** Engineering Teams | **Purpose:** Full environment replication from scratch

---

## Table of Contents

1. [Project Overview & Developer Specs](#1-project-overview--developer-specs)
   - 1.1 Project Identity
   - 1.2 Microservices Architecture
   - 1.3 System Architecture Diagram
2. [Version Control & GitOps Strategy](#2-version-control--gitops-strategy)
   - 2.1 Repository Details
   - 2.2 Branching Strategy
   - 2.3 GitHub Secrets
   - 2.4 SMTP Configuration
3. [DevOps Infrastructure — How It Works](#3-devops-infrastructure--how-it-works)
   - 3.1 Traffic Routing
   - 3.2 CI/CD Pipeline
4. [From-Scratch Setup Guide](#4-from-scratch-setup-guide)
   - 4.1 AWS EC2 Provisioning
   - 4.2 Kubernetes Cluster Initialization
   - 4.3 Core Services
   - 4.4 HAProxy — SSL Termination
   - 4.5 Observability & Tools
   - 4.6 Identity Management — Keycloak
5. [Storage & Persistence Logic](#5-storage--persistence-logic)
6. [Dashboard Exposure via kgateway](#6-dashboard-exposure-via-kgateway)
7. [ArgoCD GitOps Apps](#7-argocd-gitops-apps)
8. [Post-Deployment Verification](#8-post-deployment-verification)
9. [Operational Runbooks](#9-operational-runbooks)

---

## 1. Project Overview & Developer Specs

### 1.1 Project Identity

| Field | Value |
|-------|-------|
| **Project Name** | PayCrest LMS (Loan Management System) |
| **Purpose** | End-to-end fintech platform for loan origination, EMI management, wallet operations, and verification |
| **Frontend** | React + Vite (TypeScript) |
| **API Gateway** | Node.js (Express) |
| **Backend Services** | Python FastAPI (8 microservices) |
| **Database** | MongoDB (StatefulSet) |
| **Domain** | paycrest.online |
| **Container Registry** | Docker Hub |

---

### 1.2 Microservices Architecture

| Service | Language | Namespace | Port | Function |
|---------|----------|-----------|------|----------|
| **frontend** | React/TypeScript | pc-frontend | 80 | Customer and staff UI |
| **api-gateway** | Node.js | pc-edge | 3000 | Request routing, auth middleware |
| **auth-service** | Python FastAPI | pc-app | 8000 | User authentication, JWT, MPIN |
| **loan-service** | Python FastAPI | pc-app | 8000 | Loan origination, EMI schedule, calculations |
| **admin-service** | Python FastAPI | pc-app | 8000 | Staff management, approvals, audit |
| **manager-service** | Python FastAPI | pc-app | 8000 | Manager review and sanction |
| **verification-service** | Python FastAPI | pc-app | 8000 | KYC, document verification, scoring |
| **payment-service** | Python FastAPI | pc-app | 8000 | Cashfree payment gateway integration |
| **wallet-service** | Python FastAPI | pc-app | 8000 | Wallet, transactions, MPIN |
| **emi-service** | Python FastAPI | pc-app | 8000 | EMI penalties, monitoring, notifications |
| **mongodb** | MongoDB 6 | pc-data | 27017 | Persistent data store (StatefulSet) |

#### Required Secrets per Service

All Python services share the same secret structure via Kubernetes Secrets. The following environment variables are required:

**API Gateway (`api-gateway`):**
```
MONGODB_URI
JWT_SECRET
CASHFREE_APP_ID
CASHFREE_SECRET_KEY
AUTH_SERVICE_URL
LOAN_SERVICE_URL
ADMIN_SERVICE_URL
MANAGER_SERVICE_URL
VERIFICATION_SERVICE_URL
PAYMENT_SERVICE_URL
WALLET_SERVICE_URL
EMI_SERVICE_URL
```

**Python Microservices (common):**
```
MONGODB_URI
JWT_SECRET
SECRET_KEY
CASHFREE_APP_ID
CASHFREE_SECRET_KEY
```

**MongoDB:**
```
MONGO_INITDB_ROOT_USERNAME
MONGO_INITDB_ROOT_PASSWORD
MONGO_INITDB_DATABASE
```

---

### 1.3 System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           INTERNET / USERS                                   │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ HTTPS :443
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    HAProxy + Certbot (ip-10-0-1-27)                          │
│              Elastic IP: 23.23.104.150                                       │
│   - SSL Termination (Let's Encrypt)                                          │
│   - ACL-based subdomain routing                                              │
│   - Basic auth for monitoring dashboards                                     │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ HTTP :80 (internal)
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│              kgateway (Envoy Proxy) — NodePort                               │
│              Namespace: pc-gateway                                           │
│   - HTTPRoute-based hostname routing                                         │
│   - Routes to ALL services and dashboards                                    │
└──────┬────────────────┬──────────────────┬──────────────────┬───────────────┘
       │                │                  │                  │
       ▼                ▼                  ▼                  ▼
  [App Routes]   [Grafana Route]   [ArgoCD Route]   [Keycloak Route]
  pc-frontend    monitoring ns     argocd ns         keycloak ns
  pc-edge        monitoring ns     ...               ...
                 [Loki Route]
                 [Prometheus Route]
                 [Headlamp Route]
                 [Rollouts Route]

┌─────────────────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster (kubeadm)                              │
│  ┌──────────────────────┐  ┌──────────────────────────────────────────────┐ │
│  │   Master Node        │  │          Worker Nodes                        │ │
│  │   ip-10-0-1-155      │  │  ip-10-0-1-102 | ip-10-0-1-245 | ip-10-0-1-85│ │
│  │   Control Plane      │  │  App Pods | Dashboard Pods                  │ │
│  └──────────────────────┘  └──────────────────────────────────────────────┘ │
│                                                                               │
│  Namespaces: pc-frontend | pc-edge | pc-gateway | pc-app | pc-data          │
│              monitoring | logging | argocd | argo-rollouts                   │
│              keycloak | kube-system | kgateway-system                        │
└─────────────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                NFS Server (ip-10-0-1-27) + NFS CSI Driver                   │
│         Shared persistent storage for all PVCs                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Version Control & GitOps Strategy

### 2.1 Repository Details

| Field | Value |
|-------|-------|
| **Repository URL** | `https://github.com/Guys-Chill-Madi/PayCrest-Dev` |
| **Primary Branch** | `master` |
| **CI Branch** | `test` |
| **Backup Branch** | `backup` |

### 2.2 Branching Strategy

```
master ─────────────────────────────────────────────────► (production)
          ↑ PR + auto-merge (release pipeline)
test ────────────────────────────────────────────────────► (staging/CI)
          ↑ feature PRs
feature/* ──────────────────────────────────────────────► (dev)
```

- All PRs must pass CI pipeline before merge
- Merges to `test` trigger Docker image build and push
- Release tags (`service-name-vX.Y.Z`) trigger the release pipeline
- ArgoCD watches `master` branch and syncs automatically

### 2.3 GitHub Secrets

#### Organization-Level Secrets (shared across all repos)

| Secret Name | Description |
|-------------|-------------|
| `DOCKER_USERNAME` | Docker Hub username |
| `DOCKER_PASSWORD` | Docker Hub password/token |
| `SONAR_HOST_URL` | SonarQube server URL (https://sonar.yourdomain.com) |
| `SNYK_TOKEN` | Snyk API token for dependency scanning |
| `MAIL_USERNAME` | SMTP email address for notifications |
| `MAIL_PASSWORD` | SMTP password/app password |
| `DEVELOPMENT_TEAM_EMAIL` | Team email for release notifications |

#### Repository-Level Secrets (one per service)

| Secret Name | Service |
|-------------|---------|
| `SONAR_TOKEN_FRONTEND` | frontend |
| `SONAR_TOKEN_API_GATEWAY` | api-gateway |
| `SONAR_TOKEN_AUTH_SERVICE` | auth-service |
| `SONAR_TOKEN_LOAN_SERVICE` | loan-service |
| `SONAR_TOKEN_ADMIN_SERVICE` | admin-service |
| `SONAR_TOKEN_MANAGER_SERVICE` | manager-service |
| `SONAR_TOKEN_VERIFICATION_SERVICE` | verification-service |
| `SONAR_TOKEN_PAYMENT_SERVICE` | payment-service |
| `SONAR_TOKEN_WALLET_SERVICE` | wallet-service |
| `SONAR_TOKEN_EMI_SERVICE` | emi-service |
| `PAT_TOKEN` | GitHub Personal Access Token (repo scope) for auto-merge |

### 2.4 SMTP Configuration

For release notifications via Gmail:

1. Enable 2-Factor Authentication on your Gmail account
2. Generate an App Password: Google Account → Security → App Passwords
3. Set `MAIL_USERNAME` = your Gmail address
4. Set `MAIL_PASSWORD` = the generated 16-character app password
5. The pipeline uses `smtplib` with `smtp.gmail.com:587` (TLS)

---

## 3. DevOps Infrastructure — How It Works

### 3.1 Traffic Routing

```
User Request (HTTPS)
        │
        ▼
HAProxy (Port 443) — SSL termination
        │
        ├── ACL: sonar.yourdomain.com ──────────► SonarQube EC2 (port 9000) [direct]
        │
        └── All other subdomains ────────────────► kgateway NodePort
                                                          │
                                        ┌─────────────────┼─────────────────────┐
                                        │                 │                     │
                               HTTPRoute hostname    HTTPRoute hostname   HTTPRoute hostname
                               grafana.domain        argocd.domain        keycloak.domain
                                        │                 │                     │
                                   Grafana svc      ArgoCD svc          Keycloak svc
                                (monitoring ns)    (argocd ns)         (keycloak ns)
```

**Key Design Decisions:**
- HAProxy terminates SSL for ALL subdomains using a single wildcard/multi-domain certificate
- kgateway (Envoy proxy) handles ALL internal routing via HTTPRoutes — no NodePorts for services
- SonarQube is the only exception — it runs on a separate EC2 and is accessed directly
- Basic authentication for sensitive dashboards (Prometheus, Rollouts, Loki) is enforced at HAProxy level using ACLs

### 3.2 CI/CD Pipeline

#### Pipeline Flow

```
Developer pushes code
        │
        ▼
PR Created → GitHub Actions triggers
        │
        ├── Stage 1: Validate Labels
        │       └── PR must have correct labels to proceed
        │
        ├── Stage 2: SonarQube Analysis
        │       └── Scans for bugs, vulnerabilities, code smells
        │
        ├── Stage 3: Quality Gate
        │       └── Fails pipeline if code quality standards not met
        │
        ├── Stage 4: Snyk Security Scan
        │       └── Dependency vulnerability scanning
        │
        ├── Stage 5: Docker Build
        │       └── Builds image from Dockerfile
        │
        ├── Stage 6: Trivy Scan
        │       └── Container image vulnerability scanning
        │
        └── Stage 7: Helm Lint
                └── Validates Helm chart syntax

PR Merged to test branch
        │
        ▼
Build SHA image → Push to Docker Hub → Write SHA to .last-sha file

Release Tag Created (service-name-vX.Y.Z)
        │
        ▼
Release Pipeline:
  1. Detect service from tag
  2. Retag SHA image as versioned release
  3. Update service values.yaml with new image tag
  4. Update all environment values files
  5. Create PR → auto-merge to master
  6. Send email notification

ArgoCD detects values.yaml change in master
        │
        ▼
Argo Rollouts — Blue-Green Deployment
        │
        ├── Deploy new version as "green" (preview)
        ├── Keep current "blue" live
        ├── Team reviews green
        └── Manual promotion → instant traffic switch
```

#### Reusable Workflow Templates

| Template | Used By |
|----------|---------|
| `_ci-python.yml` | All 8 FastAPI microservices |
| `_ci-node.yml` | api-gateway |
| `_ci-frontend.yml` | frontend |
| `_ci-maintenance.yml` | scheduled maintenance |

---

## 4. From-Scratch Setup Guide

### 4.1 AWS EC2 Provisioning

#### Required Instances

| Instance | Type | Role | Storage |
|----------|------|------|---------|
| Master Node | t2.medium (2 vCPU, 4GB RAM) | Kubernetes control plane | 20GB |
| Worker Node 1 | t2.medium | Application workloads | 20GB |
| Worker Node 2 | t2.medium | Application workloads | 20GB |
| Worker Node 3 | t2.medium | Application workloads | 20GB |
| HAProxy + NFS | t2.micro | Load balancer + NFS server | 20GB |
| SonarQube | t2.medium | Code quality server | 30GB |

#### Security Group Rules

**Master Node:**
```
Inbound:
- Port 6443 (TCP) — Kubernetes API (from workers + team member IPs)
- Port 2379-2380 (TCP) — etcd (from workers)
- Port 10250 (TCP) — kubelet (from workers)
- Port 22 (SSH) — admin access
```

**Worker Nodes:**
```
Inbound:
- Port 10250 (TCP) — kubelet (from master)
- Port 30000-32767 (TCP) — NodePort range (from HAProxy)
- Port 22 (SSH) — admin access
- All traffic from within VPC
```

**HAProxy:**
```
Inbound:
- Port 80 (TCP) — HTTP (public)
- Port 443 (TCP) — HTTPS (public)
- Port 22 (SSH) — admin access
```

**SonarQube:**
```
Inbound:
- Port 9000 (TCP) — SonarQube UI (from HAProxy + admin)
- Port 22 (SSH) — admin access
```

---

### 4.2 Kubernetes Cluster Initialization

#### Step 1 — Install Kubernetes on ALL Nodes

Create and run this script on every node (master + all workers):

```bash
# Save as k8s-install.sh and run: chmod +x k8s-install.sh && sudo ./k8s-install.sh

set -euo pipefail

K8S_MINOR="v1.34"
REPO_URL="https://pkgs.k8s.io/core:/stable:/${K8S_MINOR}/deb/"
KEYRING="/etc/apt/keyrings/kubernetes-apt-keyring.gpg"

echo "[INFO] 1) Preparing OS prerequisites..."
cat <<'EOF' | tee /etc/modules-load.d/containerd.conf >/dev/null
overlay
br_netfilter
EOF

modprobe overlay || true
modprobe br_netfilter || true

cat <<'EOF' | tee /etc/sysctl.d/99-kubernetes-cri.conf >/dev/null
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system

swapoff -a || true
sed -ri 's@^([^#].*\s+swap\s+.*)$@#\1@g' /etc/fstab || true

echo "[INFO] 2) Installing base dependencies..."
apt-get update -y
apt-get install -y ca-certificates curl gpg apt-transport-https lsb-release

echo "[INFO] 3) Installing containerd..."
apt-get install -y containerd
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml >/dev/null
sed -ri 's@(^\s*SystemdCgroup\s*=\s*)false@\1true@g' /etc/containerd/config.toml
systemctl daemon-reload
systemctl enable --now containerd

echo "[INFO] 4) Configuring Kubernetes APT repository..."
mkdir -p /etc/apt/keyrings
curl -fsSL "${REPO_URL}Release.key" | gpg --dearmor -o "${KEYRING}"
chmod 0644 "${KEYRING}"
cat <<EOF | tee /etc/apt/sources.list.d/kubernetes.list >/dev/null
deb [signed-by=${KEYRING}] ${REPO_URL} /
EOF
apt-get update -y

echo "[INFO] 5) Installing kubelet, kubeadm, kubectl..."
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
systemctl disable --now kubelet || true

echo "[INFO] Done! Node is ready for kubeadm."
```

#### Step 2 — Initialize Master Node

Run these commands **on the master node only**:

```bash
# Initialize cluster
sudo kubeadm init \
  --pod-network-cidr 192.168.0.0/16 \
  --kubernetes-version 1.34.0

# Configure kubectl for the ubuntu user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Calico CNI
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml

# Verify nodes
kubectl get nodes

# Get join command for workers
kubeadm token create --print-join-command
```

#### Step 3 — Join Worker Nodes

Run the join command (from step 2 output) on **each worker node**:

```bash
# Example (use YOUR actual join command)
sudo kubeadm join <MASTER_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

#### Step 4 — Verify Cluster

```bash
# On master node
kubectl get nodes
# All nodes should show STATUS: Ready
```

#### Step 5 — Install Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/high-availability-1.21+.yaml

# Patch for insecure TLS (needed in self-signed cert environments)
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

# Verify
kubectl get pods -n kube-system -l k8s-app=metrics-server
kubectl top nodes
```

---

### 4.3 Core Services

#### NFS Server Setup

Run on the **dedicated NFS/HAProxy EC2 instance**:

```bash
sudo apt update
sudo apt install nfs-kernel-server -y

# Create NFS directories (one per service for clean separation)
sudo mkdir -p /var/nfs/paycrest
sudo mkdir -p /var/nfs/keycloak
sudo mkdir -p /var/nfs/grafana
sudo mkdir -p /var/nfs/loki
sudo mkdir -p /var/nfs/prometheus

# Set permissions
sudo chown -R nobody:nogroup /var/nfs/
sudo chmod -R 777 /var/nfs/

# Configure exports — add all worker node IPs
sudo nano /etc/exports
```

Add to `/etc/exports` (replace with your actual worker IPs):
```
/var/nfs/paycrest    10.0.1.0/24(rw,sync,no_root_squash,no_subtree_check)
/var/nfs/keycloak    10.0.1.0/24(rw,sync,no_root_squash,no_subtree_check)
/var/nfs/grafana     10.0.1.0/24(rw,sync,no_root_squash,no_subtree_check)
/var/nfs/loki        10.0.1.0/24(rw,sync,no_root_squash,no_subtree_check)
/var/nfs/prometheus  10.0.1.0/24(rw,sync,no_root_squash,no_subtree_check)
```

```bash
# Apply exports
sudo exportfs -ra
sudo systemctl enable nfs-kernel-server
sudo systemctl restart nfs-kernel-server

# Verify
sudo exportfs -v
```

#### NFS Client Setup

Run on **every worker node**:

```bash
sudo apt update
sudo apt install nfs-common -y

# Test mount (optional verification)
sudo mkdir -p /nfs/test
sudo mount <NFS_SERVER_IP>:/var/nfs/paycrest /nfs/test
df -h | grep nfs
sudo umount /nfs/test
```

#### NFS CSI Driver

Run on **master node**:

```bash
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm repo update

helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
  --namespace kube-system \
  --version 4.12.0

# Verify
kubectl get pods -n kube-system | grep csi-nfs
```

Create the StorageClass:

```yaml
# storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: <NFS_SERVER_IP>
  share: /var/nfs/paycrest
reclaimPolicy: Retain
volumeBindingMode: Immediate
mountOptions:
  - nfsvers=4.1
```

```bash
kubectl apply -f storageclass.yaml
```

#### kgateway Installation

```bash
# Install Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml

# Install kgateway CRDs
helm upgrade -i --create-namespace \
  --namespace kgateway-system \
  --version v2.3.0-main \
  kgateway-crds oci://cr.kgateway.dev/kgateway-dev/charts/kgateway-crds

# Install kgateway
helm upgrade -i -n kgateway-system \
  kgateway oci://cr.kgateway.dev/kgateway-dev/charts/kgateway \
  --version v2.3.0-main

# Verify
kubectl get pods -n kgateway-system
kubectl get gatewayclass kgateway
```

---

### 4.4 HAProxy — SSL Termination

#### Step 1 — Install HAProxy and Certbot

Run on the **HAProxy EC2 instance**:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install haproxy -y
sudo apt install snapd -y
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# Verify
haproxy -v
certbot --version
```

#### Step 2 — Configure DNS

In your DNS provider (Hostinger, Cloudflare, etc.), create A records for all subdomains pointing to your HAProxy Elastic IP:

```
yourdomain.com          → HAProxy Elastic IP
www.yourdomain.com      → HAProxy Elastic IP
grafana.yourdomain.com  → HAProxy Elastic IP
prometheus.yourdomain.com → HAProxy Elastic IP
argocd.yourdomain.com   → HAProxy Elastic IP
rollouts.yourdomain.com → HAProxy Elastic IP
keycloak.yourdomain.com → HAProxy Elastic IP
loki.yourdomain.com     → HAProxy Elastic IP
headlamp.yourdomain.com → HAProxy Elastic IP
sonar.yourdomain.com    → HAProxy Elastic IP
```

Wait 5-10 minutes for DNS propagation, then verify:
```bash
nslookup yourdomain.com
nslookup grafana.yourdomain.com
```

#### Step 3 — Obtain SSL Certificate

```bash
# Stop HAProxy to free port 80 for certbot
sudo systemctl stop haproxy

# Get certificates for all domains
sudo certbot certonly --standalone \
  -d yourdomain.com \
  -d www.yourdomain.com \
  -d grafana.yourdomain.com \
  -d prometheus.yourdomain.com \
  -d argocd.yourdomain.com \
  -d rollouts.yourdomain.com \
  -d keycloak.yourdomain.com \
  -d loki.yourdomain.com \
  -d headlamp.yourdomain.com \
  -d sonar.yourdomain.com \
  --agree-tos \
  --email your-email@gmail.com \
  --non-interactive

# Combine cert + key into single PEM for HAProxy
cat /etc/letsencrypt/live/yourdomain.com/fullchain.pem \
    /etc/letsencrypt/live/yourdomain.com/privkey.pem \
    > /etc/ssl/private/yourdomain.com.pem

chmod 600 /etc/ssl/private/yourdomain.com.pem

# Verify
ls -la /etc/ssl/private/yourdomain.com.pem
```

#### Step 4 — Configure Auto-Renewal

```bash
# Create renewal hook to recombine cert after renewal
sudo cat > /etc/letsencrypt/renewal-hooks/deploy/combine-cert.sh << 'EOF'
#!/bin/bash
cat /etc/letsencrypt/live/yourdomain.com/fullchain.pem \
    /etc/letsencrypt/live/yourdomain.com/privkey.pem \
    > /etc/ssl/private/yourdomain.com.pem
chmod 600 /etc/ssl/private/yourdomain.com.pem
systemctl reload haproxy
EOF

sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/combine-cert.sh

# Test renewal (dry run)
sudo certbot renew --dry-run
```

#### Step 5 — HAProxy Full Configuration

After getting the kgateway NodePort:
```bash
kubectl get svc -n pc-gateway paycrest-kgate
# Note the NodePort (e.g., 31748)
```

Write the full HAProxy config (replace placeholders with actual values):

```bash
sudo cat > /etc/haproxy/haproxy.cfg << 'HAPROXY'
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    maxconn 4096
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
    ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5s
    timeout client  50s
    timeout server  50s
    option  forwardfor
    option  http-server-close

# Basic auth for monitoring dashboards
userlist monitoring_users
    user admin insecure-password YourMonitoringPassword2026
    user devops insecure-password YourDevOpsPassword2026

# Redirect all HTTP to HTTPS
frontend http_redirect
    bind *:80
    mode http
    redirect scheme https code 301

# Main HTTPS frontend with ACL-based routing
frontend https_front
    bind *:443 ssl crt /etc/ssl/private/yourdomain.com.pem
    mode http
    option forwardfor
    http-response set-header Strict-Transport-Security "max-age=31536000; includeSubDomains"

    # === ACL Definitions (subdomain matching) ===
    acl host_main       hdr(host) -i yourdomain.com
    acl host_www        hdr(host) -i www.yourdomain.com
    acl host_grafana    hdr(host) -i grafana.yourdomain.com
    acl host_prometheus hdr(host) -i prometheus.yourdomain.com
    acl host_argocd     hdr(host) -i argocd.yourdomain.com
    acl host_rollouts   hdr(host) -i rollouts.yourdomain.com
    acl host_sonar      hdr(host) -i sonar.yourdomain.com
    acl host_keycloak   hdr(host) -i keycloak.yourdomain.com
    acl host_loki       hdr(host) -i loki.yourdomain.com
    acl host_headlamp   hdr(host) -i headlamp.yourdomain.com

    # === Auth protection for sensitive dashboards ===
    acl auth_ok http_auth(monitoring_users)
    http-request auth realm "PayCrest Monitoring" if host_prometheus !auth_ok
    http-request auth realm "PayCrest Monitoring" if host_rollouts !auth_ok
    http-request auth realm "PayCrest Monitoring" if host_loki !auth_ok

    # === Routing Rules ===
    # SonarQube goes directly to its EC2 (not through kgateway)
    use_backend sonar_back if host_sonar

    # All other traffic goes to kgateway (which routes via HTTPRoutes)
    default_backend kgateway_back

# === Backends ===

# kgateway — handles all app and dashboard traffic
backend kgateway_back
    mode http
    balance roundrobin
    option forwardfor
    server worker1 <WORKER1_PRIVATE_IP>:<KGATEWAY_NODEPORT> check
    server worker2 <WORKER2_PRIVATE_IP>:<KGATEWAY_NODEPORT> check
    server worker3 <WORKER3_PRIVATE_IP>:<KGATEWAY_NODEPORT> check

# SonarQube — direct to its EC2
backend sonar_back
    mode http
    balance roundrobin
    server sonar <SONARQUBE_PRIVATE_IP>:9000 check
HAPROXY

# Validate config
sudo haproxy -c -f /etc/haproxy/haproxy.cfg

# Start HAProxy
sudo systemctl start haproxy
sudo systemctl enable haproxy
sudo systemctl status haproxy
```

#### Certificate Renewal (Manual)

```bash
sudo systemctl stop haproxy

sudo certbot renew --standalone --force-renewal

cat /etc/letsencrypt/live/yourdomain.com/fullchain.pem \
    /etc/letsencrypt/live/yourdomain.com/privkey.pem \
    > /etc/ssl/private/yourdomain.com.pem

chmod 600 /etc/ssl/private/yourdomain.com.pem

sudo systemctl start haproxy
```

---

### 4.5 Observability & Tools

#### ArgoCD

```bash
# Create namespace and install
kubectl create namespace argocd

kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Download ArgoCD CLI
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# Make ArgoCD server HTTP (since HAProxy handles SSL)
kubectl patch configmap argocd-cmd-params-cm -n argocd \
  --type merge \
  -p '{"data":{"server.insecure":"true"}}'

kubectl rollout restart deployment argocd-server -n argocd

# Get initial admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d
echo

# Change password after first login
kubectl get pods -n argocd
```

#### Argo Rollouts

```bash
# Install Argo Rollouts
kubectl create namespace argo-rollouts

kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Install kubectl plugin
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x ./kubectl-argo-rollouts-linux-amd64
sudo mv ./kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts

# Verify
kubectl argo rollouts version
kubectl get pods -n argo-rollouts
```

#### Prometheus + Grafana (kube-prometheus-stack)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack (Prometheus + Grafana + AlertManager + exporters)
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=Admin@2026 \
  --set grafana.service.type=ClusterIP \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false

# Verify pods
kubectl get pods -n monitoring

# Important: Get the exact service names (they vary by chart version)
kubectl get svc -n monitoring
# Grafana service is typically: prometheus-grafana
# Prometheus service is: prometheus-kube-prometheus-prometheus
```

**Configure Grafana Root URL** (required for correct redirects behind reverse proxy):

```bash
kubectl patch deployment prometheus-grafana -n monitoring --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/env/-","value":{"name":"GF_SERVER_ROOT_URL","value":"https://grafana.yourdomain.com"}}]'
```

**Import Dashboards in Grafana UI:**

After Grafana is accessible, go to Dashboards → Import and use these IDs:

| Dashboard ID | Name |
|-------------|------|
| 1860 | Node Exporter Full |
| 12740 | Kubernetes Cluster Overview |
| 15661 | Kubernetes Pods |
| 14584 | ArgoCD |
| 15141 | Loki Logs |

#### Loki + Promtail

```bash
# Install Loki (SingleBinary mode — NFS-backed persistence)
helm install loki grafana/loki \
  --namespace logging \
  --create-namespace \
  --set deploymentMode=SingleBinary \
  --set loki.auth_enabled=false \
  --set loki.commonConfig.replication_factor=1 \
  --set loki.storage.type=filesystem \
  --set singleBinary.replicas=1 \
  --set loki.useTestSchema=true \
  --set chunksCache.enabled=false \
  --set resultsCache.enabled=false \
  --set write.replicas=0 \
  --set read.replicas=0 \
  --set backend.replicas=0 \
  --set singleBinary.persistence.enabled=true \
  --set singleBinary.persistence.storageClass=nfs-csi \
  --set singleBinary.persistence.size=10Gi

# Install Promtail (log collector — runs on every node)
helm install promtail grafana/promtail \
  --namespace logging \
  --set config.clients[0].url=http://loki-gateway.logging.svc.cluster.local/loki/api/v1/push

# Verify
kubectl get pods -n logging
kubectl get svc -n logging
```

**Add Loki as Grafana Datasource:**

1. Go to `https://grafana.yourdomain.com`
2. Left sidebar → **Connections** → **Data sources** → **Add new data source**
3. Select **Loki**
4. URL: `http://loki-gateway.logging.svc.cluster.local`
5. Authentication: **No Authentication**
6. Click **Save & Test**

#### Headlamp (Kubernetes Dashboard)

```bash
# Install from official manifest (installs to kube-system)
kubectl apply -f https://raw.githubusercontent.com/headlamp-k8s/headlamp/main/kubernetes-headlamp.yaml

# Create ServiceAccount with cluster-admin access
cat > /tmp/headlamp-sa.yaml << 'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: headlamp-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: headlamp-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: headlamp-admin
  namespace: kube-system
---
apiVersion: v1
kind: Secret
metadata:
  name: headlamp-admin-token
  namespace: kube-system
  annotations:
    kubernetes.io/service-account.name: headlamp-admin
type: kubernetes.io/service-account-token
EOF

kubectl apply -f /tmp/headlamp-sa.yaml

# Patch deployment to use the new SA
kubectl patch deployment headlamp -n kube-system --type=strategic -p '{
  "spec": {"template": {"spec": {"serviceAccountName": "headlamp-admin"}}}
}'

# Wait for rollout
kubectl rollout status deployment/headlamp -n kube-system

# Get permanent login token
kubectl get secret headlamp-admin-token -n kube-system \
  -o jsonpath='{.data.token}' | base64 -d
echo
# Save this token — paste it into Headlamp UI at headlamp.yourdomain.com
```

#### SonarQube

Run on the **dedicated SonarQube EC2 instance**:

```bash
# Install Java
sudo apt update
sudo apt install openjdk-17-jdk -y

# Install Docker
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker

# Run SonarQube as Docker container
sudo docker run -d \
  --name sonarqube \
  --restart unless-stopped \
  -p 9000:9000 \
  -v sonarqube_data:/opt/sonarqube/data \
  -v sonarqube_extensions:/opt/sonarqube/extensions \
  -v sonarqube_logs:/opt/sonarqube/logs \
  sonarqube:community

# Access at http://SONAR_EC2_IP:9000
# Default credentials: admin / admin (change on first login)
```

**SonarQube Project Setup (for each service):**

1. Login to SonarQube
2. Create Project → Manually
3. Project key: `service-name` (e.g., `auth-service`)
4. Generate token for that project
5. Add token to GitHub secrets as `SONAR_TOKEN_SERVICE_NAME`

---

### 4.6 Identity Management — Keycloak

#### Step 1 — Create NFS Directory for Keycloak

On the **NFS server**:

```bash
sudo mkdir -p /var/nfs/keycloak
sudo chown nobody:nogroup /var/nfs/keycloak
sudo chmod 777 /var/nfs/keycloak

# Add to exports
echo "/var/nfs/keycloak    10.0.1.0/24(rw,sync,no_root_squash,no_subtree_check)" \
  | sudo tee -a /etc/exports

sudo exportfs -ra
```

#### Step 2 — Create Keycloak PVC

```yaml
# keycloak-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: keycloak-data
  namespace: keycloak
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-csi
  resources:
    requests:
      storage: 2Gi
```

> **⚠️ Important:** We previously tried using NFS for Keycloak's H2 database and hit file locking issues. The fix is to use a **dedicated NFS subdirectory** (`/var/nfs/keycloak`) rather than the general NFS share, and to set the StorageClass `reclaimPolicy: Retain` so data persists across pod restarts.

#### Step 3 — Deploy Keycloak

```yaml
# keycloak.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: keycloak
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: keycloak-data
  namespace: keycloak
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-csi
  resources:
    requests:
      storage: 2Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keycloak
  namespace: keycloak
  labels:
    app: keycloak
spec:
  replicas: 1
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      securityContext:
        fsGroup: 0
        runAsUser: 0
      containers:
        - name: keycloak
          image: quay.io/keycloak/keycloak:26.1.4
          args:
            - start-dev
          env:
            - name: KC_BOOTSTRAP_ADMIN_USERNAME
              value: admin
            - name: KC_BOOTSTRAP_ADMIN_PASSWORD
              value: YourKeycloakAdminPassword2026
            - name: KC_PROXY_HEADERS
              value: xforwarded
            - name: KC_HOSTNAME_STRICT
              value: "false"
            - name: KC_HTTP_ENABLED
              value: "true"
            - name: KC_HOSTNAME
              value: https://keycloak.yourdomain.com
          ports:
            - name: http
              containerPort: 8080
          volumeMounts:
            - name: keycloak-data
              mountPath: /opt/keycloak/data
          resources:
            requests:
              memory: "512Mi"
              cpu: "200m"
            limits:
              memory: "1Gi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /realms/master
              port: 8080
            initialDelaySeconds: 90
            periodSeconds: 10
            failureThreshold: 15
      volumes:
        - name: keycloak-data
          persistentVolumeClaim:
            claimName: keycloak-data
---
apiVersion: v1
kind: Service
metadata:
  name: keycloak
  namespace: keycloak
spec:
  type: ClusterIP
  selector:
    app: keycloak
  ports:
    - port: 8080
      targetPort: 8080
```

```bash
kubectl apply -f keycloak.yaml
kubectl rollout status deployment/keycloak -n keycloak
# Wait ~90 seconds for startup
```

#### Step 4 — Configure Keycloak

1. Open `https://keycloak.yourdomain.com`
2. Login with admin / your-password
3. **Create permanent admin:**
   - Users → Create user → fill details → Email verified: ON
   - Credentials tab → Set password → Temporary: OFF
   - Role mapping → Assign `admin` realm role
   - Logout → Login as new permanent admin
4. **Create Realm:** Click `master` dropdown → Create realm → Name: `paycrest` → Create
5. **Create Client:**
   - Clients → Create client → Client type: OpenID Connect → Client ID: `kubernetes` → Next
   - Client authentication: ON → Direct access grants: ON → Standard flow: ON → Next
   - Valid redirect URIs: `http://localhost:8000`, `http://localhost:18000`
   - Web origins: `+` → Save
   - Credentials tab → copy Client Secret
6. **Add Groups Mapper:**
   - Clients → kubernetes → Client scopes → `kubernetes-dedicated` → Add mapper → By configuration → Group Membership
   - Name: `groups`, Token claim name: `groups`, Full group path: OFF, all tokens ON → Save
7. **Create Groups:** Groups → Create: `cluster-admins`, `dev-team`, `test-team`
8. **Create Users:** Users → Create each user → set password (Temporary OFF) → assign to group

#### Step 5 — Configure Kubernetes OIDC

```bash
# Backup current API server config
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml /etc/kubernetes/kube-apiserver.yaml.backup

# Add OIDC flags using Python (safe method — avoids YAML colon parsing issues)
sudo python3 << 'EOF'
with open('/etc/kubernetes/manifests/kube-apiserver.yaml', 'r') as f:
    content = f.read()

oidc_flags = """    - --oidc-issuer-url=https://keycloak.yourdomain.com/realms/paycrest
    - --oidc-client-id=kubernetes
    - --oidc-username-claim=preferred_username
    - --oidc-groups-claim=groups
    - "--oidc-username-prefix=oidc:"
    - "--oidc-groups-prefix=oidc:"
"""

content = content.replace(
    '    image: registry.k8s.io/kube-apiserver',
    oidc_flags + '    image: registry.k8s.io/kube-apiserver'
)

with open('/etc/kubernetes/manifests/kube-apiserver.yaml', 'w') as f:
    f.write(content)
print("Done")
EOF

# Validate YAML
python3 -c "import yaml; yaml.safe_load(open('/etc/kubernetes/manifests/kube-apiserver.yaml'))" && echo "YAML is valid"

# Wait for API server to restart
sleep 60
kubectl get nodes
```

#### Step 6 — Apply RBAC

```bash
cat > /tmp/oidc-rbac.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: oidc-cluster-admins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: Group
  name: "oidc:cluster-admins"
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: paycrest-readonly
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: oidc-dev-team
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: paycrest-readonly
subjects:
- kind: Group
  name: "oidc:dev-team"
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: oidc-test-team
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: paycrest-readonly
subjects:
- kind: Group
  name: "oidc:test-team"
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f /tmp/oidc-rbac.yaml
```

#### Step 7 — Team kubeconfig

Distribute this kubeconfig to team members (replace placeholders):

```yaml
# team-kubeconfig.yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    server: https://<MASTER_PUBLIC_IP>:6443
    insecure-skip-tls-verify: true
  name: cluster
contexts:
- context:
    cluster: cluster
    user: oidc-user
  name: cluster
current-context: cluster
users:
- name: oidc-user
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: kubectl
      args:
        - oidc-login
        - get-token
        - --oidc-issuer-url=https://keycloak.yourdomain.com/realms/paycrest
        - --oidc-client-id=kubernetes
        - --oidc-client-secret=<CLIENT_SECRET>
        - --grant-type=password
```

Team members also need:
```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Install kubelogin
curl -Lo kubelogin.zip https://github.com/int128/kubelogin/releases/latest/download/kubelogin_linux_amd64.zip
unzip kubelogin.zip
sudo mv kubelogin /usr/local/bin/kubectl-oidc_login
chmod +x /usr/local/bin/kubectl-oidc_login

# Save kubeconfig
mkdir -p ~/.kube
# Save team-kubeconfig.yaml content to ~/.kube/config

# Test access
kubectl get nodes
# Enter Keycloak username and password when prompted
```

---

## 5. Storage & Persistence Logic

### PVC Strategy

Every stateful component has its **own dedicated PVC**. This provides:
- Clean data isolation between services
- Independent backup and restore per service
- No data corruption risk from shared storage paths

| Service | PVC Name | Namespace | Size | NFS Path |
|---------|----------|-----------|------|----------|
| MongoDB | mongodb-data | pc-data | 20Gi | /var/nfs/paycrest |
| Grafana | grafana-pvc | monitoring | 5Gi | /var/nfs/grafana |
| Prometheus | prometheus-data | monitoring | 20Gi | /var/nfs/prometheus |
| Loki | storage-loki-0 | logging | 10Gi | /var/nfs/loki |
| Keycloak | keycloak-data | keycloak | 2Gi | /var/nfs/keycloak |

### Creating PVCs

```yaml
# All PVCs use this pattern with nfs-csi StorageClass

# Keycloak PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: keycloak-data
  namespace: keycloak
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-csi
  resources:
    requests:
      storage: 2Gi

# Grafana PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-pvc
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-csi
  resources:
    requests:
      storage: 5Gi

# Loki PVC (created automatically by Helm, but can be pre-created)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: storage-loki-0
  namespace: logging
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-csi
  resources:
    requests:
      storage: 10Gi
```

---

## 6. Dashboard Exposure via kgateway

All dashboards are exposed via kgateway HTTPRoutes — **no NodePorts** for any dashboard service.

### Architecture

```
HAProxy → kgateway (single NodePort) → HTTPRoutes → Dashboard Services (ClusterIP)
```

### Step 1 — Create the Gateway Resource

```yaml
# gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
  namespace: monitoring
spec:
  gatewayClassName: kgateway
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchExpressions:
              - key: "kubernetes.io/metadata.name"
                operator: In
                values:
                  - monitoring
                  - logging
                  - argocd
                  - argo-rollouts
                  - keycloak
                  - kube-system
                  - pc-frontend
                  - pc-edge
                  - pc-gateway
```

```bash
kubectl apply -f gateway.yaml

# Get the NodePort (use this in HAProxy config)
kubectl get svc -n monitoring | grep gateway
```

### Step 2 — Create ReferenceGrants

Each namespace needs a ReferenceGrant to allow the Gateway to route to its services:

```yaml
# reference-grants.yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-gateway
  namespace: monitoring
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: monitoring
  to:
  - group: ""
    kind: Service
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-gateway
  namespace: logging
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: logging
  to:
  - group: ""
    kind: Service
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-gateway
  namespace: argocd
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: argocd
  to:
  - group: ""
    kind: Service
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-gateway
  namespace: argo-rollouts
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: argo-rollouts
  to:
  - group: ""
    kind: Service
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-gateway
  namespace: keycloak
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: keycloak
  to:
  - group: ""
    kind: Service
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-gateway
  namespace: kube-system
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: kube-system
  to:
  - group: ""
    kind: Service
```

```bash
kubectl apply -f reference-grants.yaml
```

### Step 3 — Create HTTPRoutes for All Dashboards

```yaml
# dashboard-routes.yaml
# Replace yourdomain.com with your actual domain

# Grafana
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: grafana-route
  namespace: monitoring
spec:
  parentRefs:
  - name: main-gateway
    namespace: monitoring
  hostnames:
  - "grafana.yourdomain.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: prometheus-grafana
      port: 80
---
# Prometheus
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: prometheus-route
  namespace: monitoring
spec:
  parentRefs:
  - name: main-gateway
    namespace: monitoring
  hostnames:
  - "prometheus.yourdomain.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: prometheus-kube-prometheus-prometheus
      port: 9090
---
# ArgoCD
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: argocd-route
  namespace: argocd
spec:
  parentRefs:
  - name: main-gateway
    namespace: monitoring
  hostnames:
  - "argocd.yourdomain.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: argocd-server
      port: 80
---
# Argo Rollouts Dashboard
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: rollouts-route
  namespace: argo-rollouts
spec:
  parentRefs:
  - name: main-gateway
    namespace: monitoring
  hostnames:
  - "rollouts.yourdomain.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: rollouts-dashboard
      port: 3100
---
# Keycloak
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: keycloak-route
  namespace: keycloak
spec:
  parentRefs:
  - name: main-gateway
    namespace: monitoring
  hostnames:
  - "keycloak.yourdomain.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: keycloak
      port: 8080
---
# Headlamp
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: headlamp-route
  namespace: kube-system
spec:
  parentRefs:
  - name: main-gateway
    namespace: monitoring
  hostnames:
  - "headlamp.yourdomain.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: headlamp
      port: 80
---
# Loki
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: loki-route
  namespace: logging
spec:
  parentRefs:
  - name: main-gateway
    namespace: monitoring
  hostnames:
  - "loki.yourdomain.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: loki-gateway
      port: 80
```

```bash
kubectl apply -f dashboard-routes.yaml

# Verify all routes accepted
kubectl get httproute -A

# Check gateway attached routes count
kubectl get gateway -n monitoring main-gateway -o jsonpath='{.status.listeners[0].attachedRoutes}'
```

### Step 4 — Ensure All Dashboard Services are ClusterIP

```bash
# These should already be ClusterIP after Helm install with --set service.type=ClusterIP
# If any are NodePort, patch them:

kubectl patch svc prometheus-grafana -n monitoring \
  -p '{"spec":{"type":"ClusterIP"}}'

kubectl patch svc prometheus-kube-prometheus-prometheus -n monitoring \
  -p '{"spec":{"type":"ClusterIP"}}'

kubectl patch svc argocd-server -n argocd \
  -p '{"spec":{"type":"ClusterIP"}}'

kubectl patch svc loki-gateway -n logging \
  -p '{"spec":{"type":"ClusterIP"}}'

kubectl patch svc keycloak -n keycloak \
  -p '{"spec":{"type":"ClusterIP"}}'

kubectl patch svc headlamp -n kube-system \
  -p '{"spec":{"type":"ClusterIP"}}'
```

### Step 5 — Update HAProxy with Gateway NodePort

```bash
# Get the gateway NodePort
GATEWAY_PORT=$(kubectl get svc -n monitoring main-gateway-kgateway \
  -o jsonpath='{.spec.ports[0].nodePort}' 2>/dev/null || \
  kubectl get svc -n monitoring -o wide | grep gateway | awk '{print $5}' | cut -d: -f2 | cut -d/ -f1)

echo "Gateway NodePort: $GATEWAY_PORT"
```

Update HAProxy `kgateway_back` servers with this NodePort.

---

## 7. ArgoCD GitOps Apps

### Repository Structure for GitOps

```
your-repo/
├── argocd-apps/
│   ├── api-gateway-app.yaml
│   ├── auth-service-app.yaml
│   ├── frontend-app.yaml
│   ├── shared-infra-app.yaml
│   └── dashboards-app.yaml       ← NEW for dashboards
├── shared-infra/
│   └── Helm/
│       └── templates/
│           ├── gateway.yaml
│           ├── routes.yaml
│           └── ...
├── dashboards/                   ← NEW folder
│   └── Helm/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── keycloak.yaml
│           ├── headlamp.yaml
│           ├── rollouts-dashboard.yaml
│           ├── pvcs.yaml
│           ├── gateway.yaml
│           ├── routes.yaml
│           └── reference-grants.yaml
└── api-gateway/
    └── Helm/
        └── ...
```

### Example ArgoCD App (Dashboard Umbrella)

```yaml
# argocd-apps/dashboards-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dashboards
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR-ORG/YOUR-REPO
    targetRevision: master
    path: dashboards/Helm
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f argocd-apps/dashboards-app.yaml

# Watch sync
kubectl get application dashboards -n argocd -w
```

### Argo Rollouts Dashboard Manifest

Include this in `dashboards/Helm/templates/rollouts-dashboard.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rollouts-dashboard
  namespace: argo-rollouts
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rollouts-dashboard
  template:
    metadata:
      labels:
        app: rollouts-dashboard
    spec:
      serviceAccountName: rollouts-dashboard-sa
      containers:
        - name: dashboard
          image: quay.io/argoproj/kubectl-argo-rollouts:latest
          command:
            - kubectl-argo-rollouts
            - dashboard
            - --port=3100
          ports:
            - containerPort: 3100
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "256Mi"
              cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: rollouts-dashboard
  namespace: argo-rollouts
spec:
  type: ClusterIP
  selector:
    app: rollouts-dashboard
  ports:
    - port: 3100
      targetPort: 3100
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rollouts-dashboard-sa
  namespace: argo-rollouts
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: rollouts-dashboard-role
rules:
- apiGroups: ["argoproj.io"]
  resources: ["rollouts","rollouts/scale","rollouts/status","analysisruns","analysisruns/finalizers","experiments","analysistemplates","clusteranalysistemplates"]
  verbs: ["get","list","watch","create","update","patch","delete"]
- apiGroups: [""]
  resources: ["pods","pods/log","services","namespaces","replicationcontrollers","persistentvolumeclaims","configmaps","events"]
  verbs: ["get","list","watch","create","update","patch","delete"]
- apiGroups: ["apps"]
  resources: ["replicasets","deployments","statefulsets","daemonsets"]
  verbs: ["get","list","watch","create","update","patch","delete"]
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["get","list","watch","create","update","patch","delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: rollouts-dashboard-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: rollouts-dashboard-role
subjects:
- kind: ServiceAccount
  name: rollouts-dashboard-sa
  namespace: argo-rollouts
```

---

## 8. Post-Deployment Verification

### Verify All Services

```bash
# Cluster health
kubectl get nodes
kubectl get pods -A | grep -v Running | grep -v Completed

# Gateway routes
kubectl get httproute -A
kubectl get gateway -A

# PVCs
kubectl get pvc -A

# Check each URL
curl -I https://yourdomain.com
curl -I https://grafana.yourdomain.com
curl -I https://prometheus.yourdomain.com -u admin:YourMonitoringPassword2026
curl -I https://argocd.yourdomain.com
curl -I https://rollouts.yourdomain.com -u admin:YourMonitoringPassword2026
curl -I https://keycloak.yourdomain.com
curl -I https://headlamp.yourdomain.com
curl -I https://loki.yourdomain.com -u admin:YourMonitoringPassword2026

# Verify OIDC
kubectl get clusterrolebinding | grep oidc
kubectl auth whoami  # (after setting up team kubeconfig)
```

### Dashboard Access Summary

| URL | Login Method | Default Credentials |
|-----|-------------|---------------------|
| `grafana.yourdomain.com` | Grafana login | admin / Admin@2026 |
| `prometheus.yourdomain.com` | HAProxy basic auth | admin / YourMonitoringPassword2026 |
| `argocd.yourdomain.com` | ArgoCD login | admin / (from secret) |
| `rollouts.yourdomain.com` | HAProxy basic auth | admin / YourMonitoringPassword2026 |
| `keycloak.yourdomain.com` | Keycloak login | admin / YourKeycloakAdminPassword2026 |
| `headlamp.yourdomain.com` | Token | (from headlamp-admin-token secret) |
| `loki.yourdomain.com` | HAProxy basic auth | admin / YourMonitoringPassword2026 |
| `sonar.yourdomain.com` | SonarQube login | admin / (set on first login) |

---

## 9. Operational Runbooks

### Runbook: Cluster Restart Recovery

When EC2 instances are stopped and restarted:

```bash
# 1. Wait for all nodes to become Ready (2-5 min)
kubectl get nodes -w

# 2. Force delete stuck Unknown pods
kubectl get pods -A | grep Unknown | awk '{print "kubectl delete pod " $2 " -n " $1 " --force --grace-period=0"}' | bash

# 3. Force delete stuck Terminating pods
kubectl get pods -A | grep Terminating | awk '{print "kubectl delete pod " $2 " -n " $1 " --force --grace-period=0"}' | bash

# 4. Check all pods running
kubectl get pods -A | grep -v Running | grep -v Completed

# 5. Restart ArgoCD if needed
kubectl delete pod -n argocd -l app.kubernetes.io/name=argocd-repo-server

# 6. Verify application accessible
curl -I https://yourdomain.com
```

### Runbook: ArgoCD Repo Server Instability

ArgoCD repo-server sometimes goes into Unknown state:

```bash
kubectl delete pod -n argocd -l app.kubernetes.io/name=argocd-repo-server
kubectl get pods -n argocd -w
```

### Runbook: Certificate Renewal

```bash
# On HAProxy server
sudo systemctl stop haproxy

sudo certbot renew --standalone --force-renewal

cat /etc/letsencrypt/live/yourdomain.com/fullchain.pem \
    /etc/letsencrypt/live/yourdomain.com/privkey.pem \
    > /etc/ssl/private/yourdomain.com.pem

chmod 600 /etc/ssl/private/yourdomain.com.pem
sudo systemctl start haproxy
```

### Runbook: Blue-Green Promotion

```bash
# Check rollout status
kubectl argo rollouts get rollout <service-name> -n pc-app

# Promote to production (switch traffic from blue to green)
kubectl argo rollouts promote <service-name> -n pc-app

# Rollback if needed
kubectl argo rollouts undo <service-name> -n pc-app
```

### Runbook: Add New Team Member

1. In Keycloak (`paycrest` realm) → Users → Create user
2. Set password (Temporary: OFF)
3. Join appropriate group (`dev-team` or `test-team`)
4. Share team kubeconfig + Keycloak credentials
5. Team member installs kubectl + kubelogin
6. Tests access with `kubectl get nodes`

### Runbook: Deploy New Application Version

1. Merge feature branch to `test`
2. CI pipeline runs automatically (SonarQube → Snyk → Docker → Trivy → Helm Lint)
3. If all checks pass, Docker image tagged with SHA and pushed
4. Create GitHub Release with tag `service-name-vX.Y.Z`
5. Release pipeline runs → updates `values.yaml` → merges to master
6. ArgoCD detects change → triggers Argo Rollouts blue-green deployment
7. Review green environment → promote via `kubectl argo rollouts promote`

---

*End of PayCrest LMS Implementation Manual*

*Document maintained by: DevOps Team*
*Last updated: April 2026*
