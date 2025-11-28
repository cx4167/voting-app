# Kubernetes Deployment

Deploy the voting application to Kubernetes cluster.

## Prerequisites

- Kubernetes cluster v1.21+
- kubectl configured
- Images pulled: denish952/voting-app-{vote,worker,result}:v1

## Quick Deploy
```bash
# Create namespace and deploy all
kubectl create namespace voting-app
kubectl apply -f .

# Verify
kubectl get all -n voting-app

# Check logs
kubectl logs -n voting-app -l app=vote -f
```

## Manual Deployment
```bash
# Step 1: Create namespace
kubectl apply -f namespace.yaml

# Step 2: Configure secrets and configmaps
kubectl apply -f configmap.yaml
kubectl apply -f secrets.yaml

# Step 3: Setup storage
kubectl apply -f pv-pvc.yaml

# Step 4: Deploy services
kubectl apply -f postgres/postgres-deployment.yaml
kubectl apply -f redis/redis-deployment.yaml
kubectl apply -f worker/worker-deployment.yaml
kubectl apply -f vote/vote-deployment.yaml
kubectl apply -f result/result-deployment.yaml
```

## Accessing the Application
```bash
# Get NodePort services
kubectl get svc -n voting-app

# Access via NodePort
# Vote: http://<node-ip>:<vote-port>
# Result: http://<node-ip>:<result-port>
```

## Scaling
```bash
# Scale vote deployment to 3 replicas
kubectl scale deployment vote -n voting-app --replicas=3

# Scale worker to 2 replicas
kubectl scale deployment worker -n voting-app --replicas=2
```

## Monitoring
```bash
# Watch pods
kubectl get pods -n voting-app -w

# View pod details
kubectl describe pod <pod-name> -n voting-app

# View logs
kubectl logs -n voting-app -l app=vote --tail=50 -f

# Port forward for debugging
kubectl port-forward svc/db 5432:5432 -n voting-app
```

## Cleanup
```bash
# Delete entire namespace
kubectl delete namespace voting-app
```

## Files

- `namespace.yaml` - Kubernetes namespace
- `configmap.yaml` - Configuration maps
- `secrets.yaml` - Secrets (passwords)
- `pv-pvc.yaml` - Storage configuration
- `postgres/` - PostgreSQL deployment
- `redis/` - Redis deployment
- `vote/` - Vote service deployment
- `worker/` - Worker service deployment
- `result/` - Result service deployment
