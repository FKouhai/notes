---
id: 003-Installation
aliases:
  - argocd-install
tags:
  - argocd
  - kubernetes
  - gitops
  - installation
---

# Argo CD Installation Guide

## Prerequisites

- Kubernetes cluster (v1.16+) with kubectl configured
- Helm 3 (optional, for Helm installations)

## Installation Methods

### Method 1: Using Manifests (Recommended)

#### Install Argo CD
```bash
# Create namespace
kubectl create namespace argocd

# Apply manifests
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

#### Initial Password
```bash
# Get initial admin password
kubectl -n argocd get secret $(kubectl get sa,argocd-initial-admin-secret -o name | cut -f2 -d"/") -o jsonpath="{.data.password}" | base64 -d
```

#### Access Argo CD
Port-forward to access UI:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Access at https://localhost:8080 with username "admin" and the initial password.

### Method 2: Using Helm

#### Add Repository
```bash
# Add Argo CD Helm repository
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

#### Install with Default Values
```bash
# Create namespace
kubectl create namespace argocd

# Install Argo CD
helm install argocd argo/argo-cd --namespace argocd
```

#### Install with Custom Values
```bash
# Create values file
cat > argocd-values.yaml << EOF
redis-ha:
  enabled: true
  
controller:
  replicas: 1

server:
  service:
    type: LoadBalancer
  autoscaling:
    enabled: true
    minReplicas: 2
    
repositories:
  - url: https://github.com/yourorg/yourrepo.git
    name: your-repo
    usernameSecret:
      name: git-creds
      key: username
    passwordSecret:
      name: git-creds
      key: password
EOF

# Install with custom values
helm install argocd argo/argo-cd \
  --namespace argocd \
  --values argocd-values.yaml
```

### Method 3: Using Kustomize

#### Create kustomization.yaml
```yaml
# kustomization.yaml
resources:
- https://github.com/argoproj/argo-cd/manifests/cluster-install?ref=stable

namespace: argocd

# Optional: Patch service for LoadBalancer
patchesStrategicMerge:
- |-
  apiVersion: v1
  kind: Service
  metadata:
    name: argocd-server
  spec:
    type: LoadBalancer
```

#### Apply with Kustomize
```bash
# Apply resources
kubectl apply -k . -n argocd

# Or using kubectl v1.14+
kubectl create namespace argocd
kubectl apply -n argocd -k github.com/argoproj/argo-cd/manifests/crds?ref=stable
kubectl apply -n argocd -k github.com/argoproj/argo-cd/manifests/cluster-install?ref=stable
```

## Post-Installation Configuration

### Configure CLI
```bash
# Download CLI
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd

# Login to Argo CD server
argocd login <argocd-server>:443 --insecure
```

### Add Cluster Credentials
```bash
# Add current kubectl context
argocd cluster add <context-name>

# Add cluster with in-cluster service account
kubectl config set-context --current --namespace=argocd
argocd cluster add --in-cluster
```

### Add Git Repository
```bash
# Add repository with HTTPS
argocd repo add https://github.com/yourorg/yourrepo.git \
  --username <username> \
  --password <password>

# Add repository with SSH
argocd repo add git@github.com:yourorg/yourrepo.git \
  --ssh-private-key-path ~/.ssh/id_rsa
```

## High Availability Configuration

### With HA Mode
```bash
# Apply HA manifests instead of standard manifests
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/ha/install.yaml
```

Key differences in HA mode:
- Redis HA cluster instead of single Redis instance
- Multiple API server replicas
- Multiple repository server replicas
- Multiple application controller replicas

### Custom HA Configuration
```yaml
# values-ha.yaml for Helm
redis-ha:
  enabled: true
  exporter.enabled: true
  persistentVolume.enabled: true

controller:
  replicas: 2
  
server:
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 5
    
repoServer:
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 5
    
applicationSet:
  replicas: 2
```

## Security Configuration

### Enable TLS
```yaml
# Enable TLS for server
server:
  extraArgs:
    - --insecure=false
  ingress:
    enabled: true
    annotations:
      cert-manager.io/cluster-issuer: "letsencrypt-prod"
    hosts:
      - argocd.example.com
    tls:
      - secretName: argocd-tls
        hosts:
          - argocd.example.com
```

### Configure SSO (Dex)
```yaml
# Configure Dex for SSO with GitHub
configs:
  cm:
    dex.config: |
      connectors:
        - type: github
          id: github
          name: GitHub
          config:
            clientID: $dex.github.clientID
            clientSecret: $dex.github.clientSecret
            orgs:
            - name: your-org
```

## Verification

### Check Pod Status
```bash
# Verify all pods are running
kubectl get pods -n argocd

# Expected pods:
# argocd-application-controller-* 
# argocd-dex-server-*
# argocd-kas-kube-rbac-proxy-*
# argocd-kas-*
# argocd-notifications-controller-*
# argocd-redis-*
# argocd-repo-server-*
# argocd-server-*
```

### Test Connectivity
```bash
# Check service endpoints
kubectl get svc -n argocd

# Check if server is accessible
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Validate CLI
```bash
# Check version
argocd version

# Login and verify access
argocd login localhost:8080 --insecure --username admin --password <password>
argocd app list
```

This installation guide provides you with multiple ways to deploy Argo CD based on your infrastructure requirements, from simple single-instance deployments to production-grade HA configurations with security enhancements.