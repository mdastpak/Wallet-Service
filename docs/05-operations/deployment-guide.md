# Deployment Guide

Complete deployment instructions for the Wallet Service on Kubernetes.

See [Deployment Architecture](../01-architecture/deployment-architecture.md) for detailed Kubernetes manifests and topology.

## Quick Start

### Prerequisites
- Kubernetes 1.28+
- Helm 3.x
- kubectl configured
- Docker registry access

### Deployment Steps

1. **Create namespace**:
```bash
kubectl create namespace wallet-production
```

2. **Configure secrets**:
```bash
kubectl create secret generic wallet-db-secret \
  --from-literal=url='jdbc:oracle:thin:@//db-host:1521/wallet' \
  --from-literal=username='wallet_user' \
  --from-literal=password='${DB_PASSWORD}' \
  -n wallet-production
```

3. **Deploy application**:
```bash
kubectl apply -f k8s/deployment.yaml -n wallet-production
```

4. **Verify deployment**:
```bash
kubectl get pods -n wallet-production
kubectl logs -f deployment/wallet-service -n wallet-production
```

See complete manifests in [Deployment Architecture](../01-architecture/deployment-architecture.md).
