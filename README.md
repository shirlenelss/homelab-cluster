# Homelab Kubernetes Cluster

A GitOps-based Kubernetes cluster managed by FluxCD on a single-node Rancher environment.

## 📋 Overview

This repository contains the configuration for a homelab Kubernetes cluster with:
- **GitOps**: FluxCD for continuous deployment
- **Application**: Linkding bookmark manager
- **Monitoring**: Prometheus/Grafana stack
- **Networking**: Cloudflare Tunnel for secure external access
- **Security**: SOPS with age encryption for secrets management

## 🏗️ Architecture

```
homelab-cluster/
├── apps/
│   ├── base/linkding/          # Base Linkding configuration
│   └── staging/linkding/       # Staging overlays with secrets
├── clusters/staging/           # Flux Kustomizations
│   ├── apps.yaml              # Applications deployment
│   └── monitoring.yaml        # Monitoring stack deployment
├── monitoring/
│   ├── controllers/staging/   # Monitoring controllers
│   └── configs/staging/       # Monitoring configurations
└── infrastructure/            # Infrastructure components
```

## 🚀 Applications

### Linkding (Bookmark Manager)
- **Version**: 1.31.0
- **Namespace**: linkding
- **Access**: Via Cloudflare Tunnel
- **Storage**: 1Gi PersistentVolumeClaim (local-path)
- **Security**: Non-root user (UID 33, GID 33)

## 📦 Setup Instructions

### Prerequisites
- Kubernetes cluster (Rancher/RKE2)
- FluxCD installed
- `kubectl` and `flux` CLI tools
- SOPS with age key for secret encryption

### Initial Setup

1. **Bootstrap FluxCD**
   ```bash
   flux bootstrap git \
     --url=<your-repo-url> \
     --branch=main \
     --path=clusters/staging
   ```

2. **Configure SOPS Secret**
   ```bash
   # Create age key secret for SOPS decryption
   kubectl create secret generic sops-age \
     --namespace=flux-system \
     --from-file=age.agekey=./age.key
   ```

3. **Verify Flux Reconciliation**
   ```bash
   # Check Flux system
   flux get kustomizations
   
   # Expected kustomizations:
   # - flux-system
   # - apps (deploys linkding)
   # - monitoring-controllers
   # - monitoring-configs
   ```

### Linkding Deployment Steps

1. **Persistent Storage**
   - Uses `local-path` StorageClass (1Gi)
   - Data mounted at `/etc/linkding/data`

2. **Security Configuration**
   - Runs as non-root user (UID/GID 33)
   - fsGroup: 0 for volume ownership
   - Drops all capabilities
   - `allowPrivilegeEscalation: false`

3. **Secrets Management**
   - Encrypted with SOPS
   - Contains Linkding configuration and Cloudflare tunnel credentials

4. **Cloudflare Tunnel Setup**
   - CNAME record created: `ldpi` → tunnel
   - Cloudflared deployment with 2 replicas
   - Routes traffic from public domain to Linkding service

![Cloudflare Deployments](image-2.png)
![Cloudflare Tunnel Status](image.png)
![Linkding Access via Tunnel](image-1.png)

## 🔧 Common Operations

### Validate Kustomize Configuration
```bash
# Validate apps
kubectl kustomize apps/staging/linkding

# Build and validate
kustomize build apps/staging/linkding
```

### Manual Reconciliation
```bash
# Reconcile specific kustomization
flux reconcile kustomization apps -n flux-system

# Reconcile source
flux reconcile source git flux-system -n flux-system
```

### Debugging

**Check application status:**
```bash
kubectl get pods -n linkding
kubectl logs -n linkding -l app=linkding
```

**Check Flux status:**
```bash
flux get all -A
flux logs -n flux-system
```

**Verify secrets:**
```bash
kubectl get secrets -n linkding
```

## ⚠️ Known Issues & Solutions

### Issue: Data Loss on Pod Restart
**Problem**: Linkding bookmarks disappear after deployment restart.

**Root Cause**: PersistentVolume may not be properly bound or data directory mismatch.

**Solution**:
1. Verify PVC is bound:
   ```bash
   kubectl get pvc -n linkding
   ```
2. Check volume mount path matches Linkding's data directory
3. Ensure fsGroup/runAsUser permissions allow write access
4. For single-node clusters, verify `local-path-provisioner` is running

### Issue: Cloudflare Secret Format
**Problem**: Incorrect secret format for cloudflared credentials.

**Solution**:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tunnel-credentials
  namespace: linkding
type: Opaque
data:
  credentials.json: <base64-encoded-json>
```

Ensure JSON is properly base64-encoded:
```bash
cat tunnel-credentials.json | base64
```

## 📊 Monitoring

Access Grafana via tunnel to view:
- Pod metrics
- Resource usage
- Application logs
- Namespace overview

## 🔐 Security

- All secrets encrypted with SOPS using age encryption
- Non-root containers
- Capability dropping
- Cloudflare Tunnel for zero-trust network access
- No direct port exposure

## 🛠️ Tech Stack

- **Orchestration**: Kubernetes (Rancher/RKE2)
- **GitOps**: FluxCD v2
- **Secrets**: SOPS + age
- **Storage**: local-path-provisioner
- **Networking**: Cloudflare Tunnel
- **Monitoring**: Prometheus + Grafana
- **Service Mesh**: Istio 1.30.2 (installed, not yet configured)

## 📝 TODO

- [ ] Configure Istio Gateway for Linkding
- [ ] Implement backup strategy for PVCs
- [ ] Add alerting rules to Prometheus
- [ ] Document disaster recovery procedures
- [ ] Add more applications to the cluster
- [ ] Set up automated testing for manifests

## 🤝 Contributing

This is a personal homelab project. Feel free to use it as reference for your own setup.

## 📄 License

MIT License - See LICENSE file for details 
