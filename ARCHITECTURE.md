# E-Commerce Platform Architecture

## System Overview

The e-commerce platform is a microservices-based application deployed on Kubernetes, featuring:
- **4 application tiers** (Frontend, Backend, Database, Cache)
- **3 Kubernetes nodes** (1 control plane, 2 worker nodes)
- **Advanced scheduling** using affinity, anti-affinity, and node selectors
- **High availability** with replica management and self-healing

---

## Network Topology

```
┌─────────────────────────────────────────────────────────────┐
│                   Internet / External Access                │
│                                                              │
│  http://<NODE_IP>:30080                                     │
└──────────────────────────┬──────────────────────────────────┘
                           │ NodePort (30080)
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              Kubernetes Cluster (AWS VPC)                   │
│  172.31.0.0/20 VPC Subnet                                   │
│                                                              │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │           Control Plane (172.31.31.119)                │ │
│ │  • kube-apiserver                                       │ │
│ │  • etcd                                                 │ │
│ │  • kube-controller-manager                              │ │
│ │  • kube-scheduler                                       │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                              │
│ ┌───────────────────────────────────┌─────────────────────┐ │
│ │   Worker-Node-1 (172.31.18.239)   │ Worker-Node-2       │ │
│ │   Labels: tier=frontend, ssd       │ (172.31.23.250)     │ │
│ │                                    │ Labels: tier=backend │ │
│ │ ┌──────────────────────────────┐   │ hdd                 │ │
│ │ │ Frontend Pods (4x)           │   │ ┌─────────────────┐ │ │
│ │ │ • 192.168.212.65             │   │ │ Backend Pods     │ │
│ │ │ • 192.168.212.72             │   │ │ • 192.168.19.*   │ │
│ │ │ • 192.168.212.78             │   │ │ • 192.168.19.*   │ │
│ │ │ • 192.168.212.xx             │   │ │ • 192.168.19.*   │ │
│ │ │ (nginx:alpine)               │   │ │ (nginx:alpine)   │ │
│ │ └──────────────────────────────┘   │ └─────────────────┘ │
│ │                                    │                     │ │
│ │ ┌──────────────────────────────┐   │ ┌─────────────────┐ │ │
│ │ │ Redis Pod (1 of 2)           │   │ │ PostgreSQL Pod  │ │
│ │ │ • 192.168.212.65             │   │ │ • 192.168.19.* │ │
│ │ │ (redis:7-alpine)             │   │ │ (postgres:16)   │ │
│ │ └──────────────────────────────┘   │ └─────────────────┘ │
│ │                                    │                     │ │
│ │ ┌──────────────────────────────┐   │ ┌─────────────────┐ │ │
│ │ │ Monitoring Agent (static pod)│   │ │ Redis Pod (2 of │ │
│ │ │ • PID ns                     │   │ │ 2)              │ │
│ │ │ (static pod)                 │   │ │ • 192.168.19.*  │ │
│ │ └──────────────────────────────┘   │ └─────────────────┘ │
│ │                                    │                     │ │
│ │ System Pods:                       │ System Pods:        │ │
│ │ • calico-node                      │ • calico-node       │ │
│ │ • kube-proxy                       │ • kube-proxy        │ │
│ │ • coredns (optional)               │                     │ │
│ └───────────────────────────────────└─────────────────────┘ │
│                                                              │
│ Calico CNI: BGP Mesh Mode                                   │
│ • Native routing (no IPIP tunneling)                        │
│ • Pod CIDR: 192.168.0.0/16                                  │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## Service Communication Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                    Service Mesh (ClusterIP)                  │
│                  IP: 10.96.0.0/12 (Service CIDR)            │
│                                                               │
│  ┌──────────────────────┬──────────────────────┐             │
│  │  frontend:80 (NPort) │  backend:8080        │             │
│  │  10.105.136.71:80    │  10.97.88.240:8080   │             │
│  │  NodePort:30080      │                      │             │
│  │  ↓ pods:4            │  ↓ pods:3            │             │
│  └──────────────────────┴──────────────────────┘             │
│                         │                                     │
│         ┌───────────────┼───────────────┐                   │
│         ▼               ▼               ▼                   │
│  ┌────────────┐  ┌────────────┐  ┌──────────────┐          │
│  │ postgres   │  │   redis    │  │   Other      │          │
│  │ 10.102.*   │  │ 10.103.*   │  │   Services   │          │
│  │ port:5432  │  │ port:6379  │  │              │          │
│  │ ↓ pods:1   │  │ ↓ pods:2   │  │              │          │
│  └────────────┘  └────────────┘  └──────────────┘          │
│                                                               │
│  DNS Names (per namespace suffix):                          │
│  • backend (→ backend.ecommerce.svc.cluster.local)          │
│  • postgres (→ postgres.ecommerce.svc.cluster.local)        │
│  • redis (→ redis.ecommerce.svc.cluster.local)              │
│  • frontend (→ frontend.ecommerce.svc.cluster.local)        │
│                                                               │
│  Resolved by CoreDNS (10.96.0.10:53)                        │
└──────────────────────────────────────────────────────────────┘
```

---

## Application Tier Architecture

### 1. Frontend Tier

**Purpose:** Customer-facing web application

**Deployment Details:**
```yaml
Deployment: frontend
Replicas: 4
Image: nginx:alpine
Node Selector: tier=frontend
Node Affinity: None
Pod Affinity: None
Resources:
  CPU: 100m (request), unlimited (limit)
  Memory: 128Mi (request), unlimited (limit)
```

**Scheduling Constraints:**
- Must run on `worker-node-1` (tier=frontend label)
- Maximum 4 replicas on single node (no anti-affinity)
- Service: NodePort (30080) for external access

**Data Flow:**
```
Client Request
  ↓
kube-proxy (NodePort 30080)
  ↓
ClusterIP service (10.105.136.71:80)
  ↓
Frontend Pod (192.168.212.x:80)
  ↓
Environment: BACKEND_URL=http://backend:8080
  ↓
DNS: backend → backend.ecommerce.svc.cluster.local
```

---

### 2. Backend API Tier

**Purpose:** Business logic and data processing

**Deployment Details:**
```yaml
Deployment: backend
Replicas: 3
Image: nginx:alpine
Node Selector: tier=backend
Node Affinity:
  Required: tier=backend
  Preferred: storage=ssd (weight=100)
Resources:
  CPU: 200m (request)
  Memory: 256Mi (request)
```

**Scheduling Constraints:**
- Must run on `worker-node-2` (tier=backend label)
- All 3 replicas on same node (node selector)
- Required affinity enforced
- Service: ClusterIP (internal only)

**Data Flow:**
```
Frontend Pod
  ↓
curl http://backend:8080
  ↓
DNS Resolution → backend.ecommerce.svc.cluster.local
  ↓
ClusterIP: 10.97.88.240:8080
  ↓
Load balanced to one of 3 backend pods
  ↓
Backend Container (nginx:alpine)
```

---

### 3. Database Tier

**Purpose:** Persistent data storage

**Deployment Details:**
```yaml
Deployment: postgres
Replicas: 1
Image: postgres:16
Node Selector: tier=backend
Node Affinity: None
Service: ClusterIP (5432)
Resources:
  CPU: 500m (request), 1000m (limit)
  Memory: 512Mi (request), 1Gi (limit)
Environment:
  POSTGRES_PASSWORD: secure_password
```

**Scheduling Constraints:**
- Must run on `worker-node-2` (tier=backend)
- Single replica (no redundancy)
- No automatic failover
- Service: ClusterIP (internal only)

**Data Flow:**
```
Backend Pod
  ↓
psql connection string: postgres.ecommerce.svc.cluster.local:5432
  ↓
ClusterIP Service: 10.102.105.203:5432
  ↓
PostgreSQL Pod
  ↓
Data persisted (if volume mounted)
```

---

### 4. Cache Tier

**Purpose:** High-performance caching

**Deployment Details:**
```yaml
Deployment: redis
Replicas: 2
Image: redis:7-alpine
Pod Anti-Affinity: REQUIRED
  topologyKey: kubernetes.io/hostname
Resources:
  CPU: 250m (request)
  Memory: 256Mi (request)
Service: ClusterIP (6379)
```

**Scheduling Constraints:**
- Pod anti-affinity: Ensures 1 pod per node
- Distributed across worker-node-1 and worker-node-2
- Can tolerate both frontend and backend taints
- Service: ClusterIP (internal)

**Distribution:**
```
redis-pod-1 → worker-node-1 (192.168.212.x:6379)
redis-pod-2 → worker-node-2 (192.168.19.x:6379)
```

**Data Flow:**
```
Backend/Frontend Pod
  ↓
redis-cli -h redis:6379
  ↓
DNS: redis → redis.ecommerce.svc.cluster.local
  ↓
ClusterIP: 10.103.250.129:6379
  ↓
Load balanced to nearest redis pod
```

---

## Scheduling Strategy

### Node Labels and Taints

**Worker-Node-1 (Frontend Tier)**
```yaml
Labels:
  tier: frontend
  storage: ssd
  kubernetes.io/hostname: worker-node-1

Taints: None (untainted)
```

**Worker-Node-2 (Backend Tier)**
```yaml
Labels:
  tier: backend
  storage: hdd
  kubernetes.io/hostname: worker-node-2

Taints: None (untainted)
```

### Pod Placement Rules

**Frontend:**
- ✅ nodeSelector: tier=frontend → worker-node-1 ONLY
- ✅ No affinity rules → all 4 on same node
- ❌ Cannot run on worker-node-2

**Backend:**
- ✅ nodeSelector: tier=backend → worker-node-2 ONLY
- ✅ Required affinity: tier=backend
- ✅ Preferred affinity: storage=ssd (but worker-node-2 has hdd)
- ❌ Cannot run on worker-node-1

**Redis:**
- ✅ Pod anti-affinity: REQUIRED (1 per node)
- ✅ Tolerations: both frontend and backend taints
- ✅ Distribution: split between nodes

**PostgreSQL:**
- ✅ nodeSelector: tier=backend → worker-node-2 ONLY
- ✅ Tolerations: backend taint
- ✅ Single replica (no HA)

---

## Control Flow and Pod Communication

### Request Path Example

**Frontend to Backend with data persistence:**

```
1. Client Request
   HTTP GET http://172.31.18.239:30080/data
   ↓
2. NodePort Service
   kube-proxy receives on port 30080
   Translates to ClusterIP:80
   ↓
3. Frontend Pod (worker-node-1)
   Receives request
   Reads from environment: BACKEND_URL=http://backend:8080
   ↓
4. DNS Query
   nslookup backend
   Resolves to 10.97.88.240:8080
   ↓
5. Backend Service (ClusterIP)
   Routes to 3 backend pods (load balanced)
   ↓
6. Backend Pod (worker-node-2)
   Processes request
   Queries cache (redis) first
   ↓
7. Redis Service
   Checks redis pods (one on each node)
   ↓
8. Redis Pod
   Returns cached data or miss
   ↓
9. PostgreSQL Query
   If cache miss, query postgres
   ↓
10. PostgreSQL Service & Pod
    Returns data from database
    ↓
11. Redis Update
    Frontend caches result
    ↓
12. Response
    Backend → Frontend → Client
```

---

## Networking Infrastructure

### Calico CNI Configuration

**Pod CIDR:**
```
192.168.0.0/16 (32,000+ pod IPs)
```

**Node-to-Pod IP Allocation:**
```
worker-node-1: 192.168.212.0/26 (64 IPs)
  • Frontend pods: .65-.127
  • Redis pod: .65
  • Test pods: .72, .78, etc.

worker-node-2: 192.168.19.128/26 (64 IPs)
  • Backend pods: .131-.133
  • PostgreSQL pod: .129
  • Redis pod: .130
```

**Service CIDR:**
```
10.96.0.0/12 (1M+ service IPs)
```

**Routing:**
```
BGP Mesh Mode (Native Routing)
  • No IPIP tunneling (disabled)
  • Direct IP routes via kernel routing table
  • AWS src/dst checks disabled
  • Calico announces pod CIDR routes via BGP
  • Bird daemon manages routing
```

---

## Security and Isolation

### Namespace Isolation
```yaml
Namespace: ecommerce
  • All application pods
  • All services
  • All configurations
  • Separate from kube-system
```

### No Network Policies
```yaml
No restrictions between pods
  • All pods can communicate with all other pods
  • Frontend can query any service
  • Backend can reach cache and database
  • Open internal network (typical for private clusters)
```

### Pod Security
```yaml
- Image: Public images (no authentication required)
- Security Context: Default (no specific restrictions)
- RBAC: Default Kubernetes RBAC
- No pod security policies configured
```

---

## Data Flow Diagrams

### Synchronous Data Flow (Request/Response)

```
┌─────────────────┐
│   Client        │
└────────┬────────┘
         │ HTTP GET /data
         ▼
┌──────────────────────────┐
│ NodePort 30080           │
│ (kube-proxy iptables)    │
└────────┬─────────────────┘
         │ → 10.105.136.71:80
         ▼
┌──────────────────────────┐
│ Frontend Pod (4 replicas)│
│ Load balanced            │
└────────┬─────────────────┘
         │ curl http://backend:8080
         ▼
┌──────────────────────────┐
│ DNS Query (CoreDNS)      │
│ backend.ecommerce.*      │
└────────┬─────────────────┘
         │ → 10.97.88.240:8080
         ▼
┌──────────────────────────┐
│ Backend Pod (3 replicas) │
│ Load balanced            │
└────────┬─────────────────┘
         │ Check Redis first
         ▼
┌──────────────────────────┐
│ Redis Service            │
│ Distributed (2 pods)     │
└────────┬─────────────────┘
         │ MISS?
         ▼
┌──────────────────────────┐
│ PostgreSQL Service       │
│ Single Pod               │
└────────┬─────────────────┘
         │ Query DB
         ▼
┌──────────────────────────┐
│ Response Data            │
│ Back through stack       │
└─────────────────────────┘
```

---

## High Availability Features

### Pod-Level HA
- **Frontend:** 4 replicas (stateless)
- **Backend:** 3 replicas (stateless)
- **Redis:** 2 replicas (distributed caching)
- **PostgreSQL:** 1 replica (single point of failure)

### Node-Level HA
- **2 worker nodes** (Redis distributed)
- **1 control plane** (not HA)

### Service-Level HA
- **LoadBalancer:** Frontend (NodePort)
- **Internal LB:** ClusterIP (round-robin)
- **DNS:** CoreDNS (2 replicas in kube-system)

### Self-Healing
- **ReplicaSet:** Maintains desired pod count
- **kubelet:** Restarts failed containers
- **Calico:** Maintains network connectivity

### Failure Scenarios
```
Frontend Pod fails
  → ReplicaSet creates new pod → Service reroutes traffic

Redis Pod fails
  → Anti-affinity ensures other pod unaffected
  → Cache miss increases, performance degrades

Backend Pod fails
  → ReplicaSet recreates → Service reroutes

PostgreSQL Pod fails
  → Data loss (no replica)
  → Application errors (if not cached)

Worker Node fails
  → Pods evicted (depending on affinity)
  → Redis: Other pod continues (distributed)
  → Backend: Reschedules (if not node-pinned)
  → Frontend: Stuck in Pending (requires frontend node)
```

---

## Performance Characteristics

### Latency
- **Frontend response:** ~2-5ms (nginx processing)
- **Backend query:** ~1-3ms (nginx processing)
- **DNS lookup:** ~3ms (CoreDNS)
- **Redis query:** <1ms (in-memory cache)
- **PostgreSQL query:** ~2-10ms (network + DB)

### Throughput
- **Nginx pod:** ~100-1000 req/s (depending on content)
- **Frontend service:** 4 pods × ~250 req/s = ~1000 req/s
- **Backend service:** 3 pods × ~200 req/s = ~600 req/s

### Resource Usage
- **Frontend:** ~2-5m CPU, ~5-10Mi memory per pod
- **Backend:** ~4-8m CPU, ~8-15Mi memory per pod
- **PostgreSQL:** ~5-15m CPU, ~40-60Mi memory
- **Redis:** ~1-3m CPU, ~10-15Mi memory per pod

---

## Scalability

### Horizontal Scaling
- **Frontend:** Up to 10+ replicas possible (node capacity)
- **Backend:** Limited to node-2 capacity (node affinity)
- **Redis:** Max 2 replicas (2 nodes, anti-affinity)
- **PostgreSQL:** Single replica (no HA)

### Vertical Scaling
- **Nodes:** Can add more nodes to cluster
- **Resources:** Can increase resource limits per pod

### Improvement Opportunities
- [ ] Add PostgreSQL replica (master-slave or HA)
- [ ] Add 3rd worker node (enables more backend pods)
- [ ] Implement HPA (Horizontal Pod Autoscaler)
- [ ] Add persistent volumes for database
- [ ] Implement caching headers for frontend

---

## Production Readiness Checklist

- [x] Multi-replica deployments
- [x] Service discovery (DNS)
- [x] Node-level scheduling rules
- [x] Resource requests/limits
- [x] Graceful termination
- [ ] Liveness/readiness probes
- [ ] Persistent storage
- [ ] Network policies
- [ ] Pod security policies
- [ ] RBAC rules
- [ ] Resource quotas
- [ ] Monitoring/observability
- [ ] Logging
- [ ] Backup/disaster recovery

