# Deployment Guide

## Prerequisites

### Cluster Requirements
- Kubernetes 1.35.0 or higher
- 3 nodes total: 1 control plane + 2 worker nodes
- All nodes running on same AWS VPC subnet
- Network connectivity between all nodes enabled

### AWS Configuration (Already Completed)
- ✅ Source/Destination checks disabled on all EC2 instances
- ✅ Security group allows all traffic between instances
- ✅ Calico CNI configured with BGP mesh mode
- ✅ IPIP tunneling disabled for native routing

### Node Preparation

#### 1. Label Worker Nodes
```bash
# Label worker-node-1 as frontend tier
kubectl label nodes worker-node-1 tier=frontend storage=ssd

# Label worker-node-2 as backend tier
kubectl label nodes worker-node-2 tier=backend storage=hdd
```

Verify labels:
```bash
kubectl get nodes --show-labels
```

#### 2. Verify Node Readiness
```bash
kubectl get nodes
# Expected output:
# NAME            STATUS   ROLES           AGE
# control-plane   Ready    control-plane   ...
# worker-node-1   Ready    <none>          ...
# worker-node-2   Ready    <none>          ...
```

## Deployment Steps

### Phase 1: Foundation (Namespace & Services)

#### Step 1: Create Namespace
```bash
kubectl create namespace ecommerce
kubectl config set-context --current --namespace=ecommerce
```

Or apply from file:
```bash
kubectl apply -f manifests/namespace.yaml
```

### Phase 2: Data Layer (Database & Cache)

#### Step 2: Deploy PostgreSQL
```bash
kubectl apply -f manifests/postgres/postgres-deployment.yaml
kubectl apply -f manifests/postgres/postgres-service.yaml

# Verify deployment
kubectl get pods -n ecommerce -l app=postgres -o wide
# Expected: 1 pod on worker-node-2 (backend tier)

# Check pod status
kubectl describe pod postgres-* -n ecommerce
```

**Expected Result:**
- Pod scheduled on **worker-node-2** (has `tier=backend` label)
- Status: Running
- Port 5432 exposed via ClusterIP service

#### Step 3: Deploy Redis
```bash
kubectl apply -f manifests/redis/redis-deployment.yaml
kubectl apply -f manifests/redis/redis-service.yaml

# Verify deployment
kubectl get pods -n ecommerce -l app=redis -o wide
# Expected: 2 pods on different nodes (pod anti-affinity)

# Check anti-affinity is working
kubectl describe pod redis-* -n ecommerce | grep -i affinity
```

**Expected Result:**
- 2 pods running on separate nodes (worker-node-1 and worker-node-2)
- Status: Running
- Replicas separated due to pod anti-affinity rules

### Phase 3: Application Layer (Backend & Frontend)

#### Step 4: Deploy Backend API
```bash
kubectl apply -f manifests/backend/backend-deployment.yaml
kubectl apply -f manifests/backend/backend-service.yaml

# Verify deployment
kubectl get pods -n ecommerce -l app=backend -o wide
# Expected: 3 pods on worker-node-2 (requires backend tier)

# Check node affinity configuration
kubectl describe deployment backend -n ecommerce | grep -A 20 "Affinity"
```

**Expected Result:**
- 3 replicas running on **worker-node-2**
- Node affinity requirement: `tier=backend`
- Service exposes port 8080 (targets pod port 80)

#### Step 5: Deploy Frontend
```bash
kubectl apply -f manifests/frontend/frontend-deployment.yaml
kubectl apply -f manifests/frontend/frontend-service.yaml

# Verify deployment
kubectl get pods -n ecommerce -l app=frontend -o wide
# Expected: 4 pods on worker-node-1 (frontend tier)

# Check NodePort service
kubectl get svc frontend -n ecommerce
# Expected: NodePort 30080
```

**Expected Result:**
- 4 replicas running on **worker-node-1**
- Node selector: `tier=frontend`
- NodePort: 30080 accessible from outside cluster

### Phase 4: Advanced Features

#### Step 6: Deploy Priority Classes
```bash
kubectl apply -f manifests/priority/priority-classes.yaml

# Verify priority classes
kubectl get priorityclass
# Expected: high-priority, low-priority classes
```

#### Step 7: Deploy Batch Processor (Low Priority)
```bash
kubectl apply -f manifests/batch/batch-job-deployment.yaml

# Verify deployment
kubectl get pods -n ecommerce -l app=batch -o wide
```

#### Step 8: Deploy Static Pod (Monitoring Agent)

**Option A: Via SSH (Recommended)**
```bash
# SSH into worker-node-1
ssh ubuntu@<worker-node-1-ip>

# Find kubelet static pod directory
sudo cat /var/lib/kubelet/kubeconfig.yaml | grep "staticPodPath"
# Typically: /etc/kubernetes/manifests

# Create monitoring agent
sudo cp manifests/monitoring/monitoring-agent.yaml /etc/kubernetes/manifests/

# Verify via kubectl
kubectl get pods -n kube-system -o wide | grep monitoring-agent
```

**Option B: Via kubectl exec**
```bash
# Note: Only works if node filesystem is accessible
kubectl debug node/worker-node-1 -it --image=ubuntu:latest
# Then create the manifest in the static pod directory
```

## Verification Checklist

### 1. Pods Running
```bash
kubectl get pods -n ecommerce -o wide
```

**Expected output:**
```
NAME                        READY   STATUS    RESTARTS   AGE   IP              NODE
backend-67cf8cc4d9-*        1/1     Running   0          5m    192.168.19.*    worker-node-2
postgres-c785b754c-*        1/1     Running   0          10m   192.168.19.*    worker-node-2
redis-5ff6b64866-*          1/1     Running   0          8m    192.168.212.*   worker-node-1
redis-5ff6b64866-*          1/1     Running   0          8m    192.168.19.*    worker-node-2
frontend-67cf8cc4d9-*       1/1     Running   0          3m    192.168.212.*   worker-node-1
frontend-67cf8cc4d9-*       1/1     Running   0          3m    192.168.212.*   worker-node-1
frontend-67cf8cc4d9-*       1/1     Running   0          3m    192.168.212.*   worker-node-1
frontend-67cf8cc4d9-*       1/1     Running   0          3m    192.168.212.*   worker-node-1
```

### 2. Services Created
```bash
kubectl get svc -n ecommerce
```

**Expected output:**
```
NAME       TYPE       CLUSTER-IP       PORT(S)           AGE
backend    ClusterIP  10.97.88.240     8080/TCP          5m
postgres   ClusterIP  10.102.105.203   5432/TCP          10m
redis      ClusterIP  10.103.250.129   6379/TCP          8m
frontend   NodePort   10.105.136.71    80:30080/TCP      3m
```

### 3. Service Discovery
```bash
# Test DNS resolution
kubectl run -it --rm debug --image=busybox --restart=Never -n ecommerce -- sh

# Inside the pod, test:
nslookup backend
nslookup postgres
nslookup redis
nslookup frontend
```

### 4. Network Connectivity
```bash
# From within a pod, test backend connectivity
kubectl exec -it <backend-pod> -n ecommerce -- sh
# Inside: curl http://postgres:5432  (should timeout or show DB response)
# Inside: curl http://redis:6379     (should timeout or show Redis response)
```

## Troubleshooting

### Problem: Pod stuck in Pending
```bash
kubectl describe pod <pod-name> -n ecommerce
```

**Common causes:**
- Node selector doesn't match: Check node labels with `kubectl get nodes --show-labels`
- Insufficient resources: Check node capacity with `kubectl describe node <node-name>`
- Toleration missing: Add tolerance for node taints

**Fix:**
```bash
# If resource issue
kubectl delete deployment <deployment> -n ecommerce
# Reduce resource requests in YAML file
kubectl apply -f manifests/<service>/

# If node selector issue
kubectl label nodes <node> <key>=<value>
```

### Problem: Service not accessible
```bash
# Check service endpoints
kubectl get endpoints <service-name> -n ecommerce

# Check pod logs
kubectl logs <pod-name> -n ecommerce

# Verify service is correctly configured
kubectl describe svc <service-name> -n ecommerce
```

### Problem: Pod not on expected node
```bash
# Check pod affinity/anti-affinity
kubectl describe pod <pod-name> -n ecommerce | grep -A 30 "Affinity"

# Check node taints and tolerations
kubectl describe node <node-name> | grep -E "Taints|Labels"
```

## Post-Deployment

### Access Frontend Application
```bash
# Get node IP
kubectl get nodes -o wide

# Access via curl
curl http://<WORKER_NODE_1_IP>:30080

# Or from within cluster
kubectl run -it --rm curl --image=curlimages/curl --restart=Never -- curl http://frontend.ecommerce.svc.cluster.local
```

### Monitor Resource Usage
```bash
# Node resources
kubectl top nodes

# Pod resources
kubectl top pods -n ecommerce

# Detailed pod info
kubectl describe pod <pod-name> -n ecommerce
```

### View Logs
```bash
# Pod logs
kubectl logs <pod-name> -n ecommerce

# Follow logs
kubectl logs -f <pod-name> -n ecommerce

# Multi-pod logs
kubectl logs -l app=backend -n ecommerce
```

## Scaling Operations

### Scale Frontend
```bash
# Scale to 6 replicas
kubectl scale deployment frontend --replicas=6 -n ecommerce

# Watch scaling
kubectl get pods -n ecommerce -l app=frontend -w
```

### Scale Backend
```bash
# Scale to 5 replicas
kubectl scale deployment backend --replicas=5 -n ecommerce

# Note: All will be scheduled on worker-node-2 due to node selector
```

## Cleanup

### Delete entire deployment
```bash
# Delete all resources in namespace
kubectl delete namespace ecommerce

# Or delete individually
kubectl delete -f manifests/ -n ecommerce
```

### Remove static pod
```bash
# SSH into worker-node-1
ssh ubuntu@<worker-node-1-ip>

# Remove static pod manifest
sudo rm /etc/kubernetes/manifests/monitoring-agent.yaml

# Kubelet will automatically remove the pod
```

## Best Practices Applied

1. **Namespace isolation**: Dedicated `ecommerce` namespace
2. **Resource management**: CPU and memory requests/limits set
3. **Health checks**: Ready for liveness/readiness probes
4. **Service discovery**: DNS-based pod-to-pod communication
5. **Scheduling**: Node selectors, affinity rules, tolerations
6. **High availability**: Multi-replica deployments, anti-affinity
7. **Environment separation**: Tier-based node labeling
8. **Monitoring**: Static pod for centralized monitoring

## Network Architecture

```
Internet
   ↓
NodePort (30080) on worker-node-1
   ↓
frontend (4 replicas on worker-node-1)
   ↓
ClusterIP services
   ├→ backend (3 replicas on worker-node-2)
   ├→ postgres (1 replica on worker-node-2)
   └→ redis (2 replicas distributed)
```

## Monitoring & Observability

Static pod `/etc/kubernetes/manifests/monitoring-agent.yaml` collects metrics from:
- Cluster components
- Node resources
- Pod events
- Service health
