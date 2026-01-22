# Kubernetes E-Commerce Platform Deployment

## Project Overview

This project demonstrates a comprehensive multi-tier e-commerce platform deployment on Kubernetes, covering advanced scheduling features, resource management, and troubleshooting.

## Architecture

The platform consists of five main components:

- **Frontend**: Customer-facing web application (4 replicas, NodePort: 30080)
- **Backend API**: Business logic and data processing (3 replicas)
- **Database**: PostgreSQL for persistent data storage (1 replica)
- **Cache**: Redis for caching layer (2 replicas)
- **Monitoring**: Static pod for monitoring agent

## Directory Structure

```
kubernetes-assignment/
├── README.md                          # This file
├── DEPLOYMENT.md                      # Deployment instructions
├── ANSWERS.md                         # Answers to all assignment questions
├── TEST_REPORT.md                     # Comprehensive testing results
├── ARCHITECTURE.md                    # Architecture design document
└── manifests/
    ├── namespace.yaml                 # ecommerce namespace
    ├── postgres/
    │   ├── postgres-deployment.yaml
    │   └── postgres-service.yaml
    ├── redis/
    │   ├── redis-deployment.yaml
    │   └── redis-service.yaml
    ├── backend/
    │   ├── backend-deployment.yaml
    │   └── backend-service.yaml
    ├── frontend/
    │   ├── frontend-deployment.yaml
    │   └── frontend-service.yaml
    ├── monitoring/
    │   └── monitoring-agent.yaml       # Static pod
    ├── priority/
    │   └── priority-classes.yaml
    ├── batch/
    │   └── batch-job-deployment.yaml
    └── broken-app.yaml                # Fixed broken deployment
```

## Quick Start

### Prerequisites
- 3-node Kubernetes cluster (1 control plane, 2 worker nodes)
- kubectl installed and configured
- Worker nodes labeled with `tier` labels (frontend/backend)
- Appropriate taints configured

### Deployment Steps

1. **Create namespace:**
   ```bash
   kubectl apply -f manifests/namespace.yaml
   ```

2. **Deploy core services:**
   ```bash
   kubectl apply -f manifests/postgres/
   kubectl apply -f manifests/redis/
   kubectl apply -f manifests/backend/
   kubectl apply -f manifests/frontend/
   ```

3. **Deploy monitoring and priority classes:**
   ```bash
   kubectl apply -f manifests/monitoring/
   kubectl apply -f manifests/priority/
   ```

4. **Verify deployment:**
   ```bash
   kubectl get pods -n ecommerce -o wide
   kubectl get svc -n ecommerce
   ```

## Accessing the Application

### Frontend (NodePort)
```bash
# Get worker node IP
kubectl get nodes -o wide

# Access via browser or curl
curl http://<WORKER_NODE_IP>:30080
```

### Service Discovery (from within cluster)
```bash
# Test DNS resolution
kubectl exec -it <pod-name> -n ecommerce -- nslookup backend
kubectl exec -it <pod-name> -n ecommerce -- nslookup postgres
kubectl exec -it <pod-name> -n ecommerce -- nslookup redis
```

## Key Kubernetes Features Implemented

### 1. Node Labeling & Selectors
- `tier=frontend` and `tier=backend` labels for environment separation
- Node selectors to constrain pod placement

### 2. Taints & Tolerations
- Taints on specialized nodes for workload separation
- Tolerations in deployments to allow scheduling on tainted nodes

### 3. Pod Affinity Rules
- Pod anti-affinity for Redis to ensure distribution across nodes
- Node affinity for Backend API (required + preferred)

### 4. Resource Management
- CPU and memory requests for all deployments
- Resource limits to prevent node overload

### 5. Service Discovery
- ClusterIP services for internal communication
- Service DNS names (service.namespace.svc.cluster.local)

### 6. Static Pods
- Monitoring agent deployed as static pod on worker-node-1
- Managed directly by kubelet, not API server

### 7. Replica Management
- Different replica counts based on workload requirements
- Self-healing through ReplicaSet controllers

### 8. Priority & Preemption
- High-priority frontend pods
- Low-priority batch processing jobs
- Preemption behavior on resource contention

## Testing

See TEST_REPORT.md for comprehensive testing results covering:
- Service discovery
- Load balancing
- Self-healing
- Scaling operations

## Troubleshooting

Common issues and solutions are documented in DEPLOYMENT.md.

For detailed pod placement and scheduling decisions, see ANSWERS.md.

## Team Responsibilities

While all team members should understand the entire system:

- **Infrastructure Lead**: Node setup, labeling, documentation
- **Database & Backend Lead**: PostgreSQL, Backend API, connectivity testing
- **Frontend & Monitoring Lead**: Redis, Frontend, Static pods
- **All Members**: Testing, troubleshooting, architecture diagram

## Documentation References

- **DEPLOYMENT.md**: Step-by-step deployment and configuration guide
- **ANSWERS.md**: Detailed answers to all assignment questions
- **TEST_REPORT.md**: Test execution and results
- **ARCHITECTURE.md**: System design and scheduling decisions

## Useful Commands

```bash
# Get pod placement
kubectl get pods -n ecommerce -o wide

# Debug pod issues
kubectl describe pod <pod-name> -n ecommerce
kubectl logs <pod-name> -n ecommerce

# Check node status
kubectl get nodes
kubectl describe node <node-name>

# Test service discovery
kubectl run -it --rm debug --image=busybox --restart=Never -- sh
# Inside pod: nslookup service-name

# Monitor resources
kubectl top nodes
kubectl top pods -n ecommerce
```

## Notes

- The broken-app.yaml has been fixed to have realistic resource requests
- All static pods are configured in /etc/kubernetes/manifests/
- The monitoring agent runs as a static pod on worker-node-1
- Services use ClusterIP by default, except Frontend which uses NodePort
