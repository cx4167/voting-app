# Kubernetes Deployment Guide

Detailed guide for deploying to Kubernetes.

## Prerequisites

1. Running Kubernetes cluster
2. kubectl access to cluster
3. Docker images on registry
```bash
kubectl cluster-info
kubectl get nodes
```

## Step-by-Step Deployment

### 1. Create Namespace
```bash
kubectl create namespace voting-app
```

### 2. Apply ConfigMaps and Secrets
```bash
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secrets.yaml
```

### 3. Create Storage
```bash
# For local development/single node
kubectl apply -f k8s/pv-pvc.yaml

# For managed clusters (AWS, GCP, Azure), use StorageClass instead
```

### 4. Deploy Database
```bash
kubectl apply -f k8s/postgres/postgres-deployment.yaml

# Wait for PostgreSQL to be ready
kubectl wait --for=condition=ready pod -l app=postgres -n voting-app --timeout=300s
```

### 5. Deploy Cache Layer
```bash
kubectl apply -f k8s/redis/redis-deployment.yaml
```

### 6. Deploy Worker
```bash
kubectl apply -f k8s/worker/worker-deployment.yaml
```

### 7. Deploy Vote & Result Services
```bash
kubectl apply -f k8s/vote/vote-deployment.yaml
kubectl apply -f k8s/result/result-deployment.yaml
```

### 8. Verify Deployment
```bash
kubectl get all -n voting-app
kubectl get svc -n voting-app
```

## Access Application
```bash
# Get NodePort
kubectl get svc vote-service -n voting-app

# Access
# http://<node-ip>:<nodeport>
```

## Troubleshooting

### Pod not starting
```bash
kubectl describe pod <pod-name> -n voting-app
kubectl logs <pod-name> -n voting-app
```

### Connection issues
```bash
kubectl exec -it <pod> -n voting-app -- /bin/sh
# Inside pod:
ping redis
ping db
```

### View worker processing
```bash
kubectl logs -n voting-app -l app=worker -f
```
