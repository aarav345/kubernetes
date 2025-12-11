# Kubernetes Lesson 4: Services & Networking

## Table of Contents
- [Overview](#overview)
- [Theory](#theory)
  - [The Networking Problem](#the-networking-problem)
  - [What are Services?](#what-are-services)
  - [Service Types](#service-types)
  - [Service Discovery](#service-discovery)
- [Practical Labs](#practical-labs)
- [Common Commands](#common-commands)
- [Key Takeaways](#key-takeaways)
- [Homework](#homework)
- [Troubleshooting](#troubleshooting)

---

## Overview

This lesson covers Kubernetes networking and Services - how to expose applications and enable communication between Pods.

**Learning Objectives:**
- Understand why Pods need Services for networking
- Learn the four Service types: ClusterIP, NodePort, LoadBalancer, ExternalName
- Master service discovery using DNS
- Practice exposing applications internally and externally
- Understand load balancing and endpoints

---

## Theory

### The Networking Problem

**Problem: Pod IPs are ephemeral and change when pods restart**

```bash
kubectl get pods -o wide
# nginx-abc   1/1   Running   0   5m   10.244.0.5

kubectl delete pod nginx-abc
kubectl get pods -o wide
# nginx-xyz   1/1   Running   0   5s   10.244.0.12  ← NEW IP!
```

**Challenges:**
- ❌ Pod IPs change on restart
- ❌ Multiple pod replicas - which one to connect to?
- ❌ No load balancing
- ❌ No service discovery
- ❌ Can't expose apps outside cluster

### What are Services?

A **Service** is an abstraction that provides:
- ✅ Stable IP address (doesn't change)
- ✅ Stable DNS name (e.g., `my-service`)
- ✅ Load balancing across pods
- ✅ Service discovery
- ✅ External access options

**Service Architecture:**
```
                    ┌─────────────┐
     Clients   ───► │   Service   │ ◄─── Stable IP: 10.96.0.10
                    │  (nginx-svc)│      DNS: nginx-svc
                    └─────────────┘
                           │
            ┌──────────────┼──────────────┐
            │              │              │
            ▼              ▼              ▼
       ┌────────┐     ┌────────┐     ┌────────┐
       │ Pod 1  │     │ Pod 2  │     │ Pod 3  │
       │10.244  │     │10.244  │     │10.244  │
       │.0.5    │     │.0.6    │     │.0.7    │
       └────────┘     └────────┘     └────────┘
        app=nginx      app=nginx      app=nginx
```

**How Services Work:**
1. Service uses a **selector** to find Pods (e.g., `app: nginx`)
2. Service automatically tracks Pod IPs as **Endpoints**
3. Service load balances traffic across healthy Pods
4. Service IP and DNS name remain stable

### Service Types

#### 1. ClusterIP (Default)

**Purpose:** Internal cluster communication only

```
┌─────────────────────────────────┐
│         Kubernetes Cluster      │
│                                 │
│  ┌──────┐        ┌──────────┐  │
│  │ Pod  │   ───► │ Service  │  │
│  │      │        │ClusterIP │  │
│  └──────┘        └──────────┘  │
│                       │         │
│                   Load Balance  │
│                       ▼         │
│              ┌────┬────┬────┐  │
│              │Pod │Pod │Pod │  │
│              └────┴────┴────┘  │
└─────────────────────────────────┘
        ▲
        │
   ❌ NOT accessible from outside
```

**Use Cases:**
- Microservice communication (frontend → backend)
- Database connections
- Internal APIs

**Example:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP  # Default, can be omitted
  selector:
    app: backend
  ports:
  - port: 8080        # Service port
    targetPort: 8080  # Pod port
```

#### 2. NodePort

**Purpose:** Expose service on each Node's IP at a static port

```
┌─────────────────────────────────┐
│         Kubernetes Cluster      │
│                                 │
│  ┌──────────┐    ┌──────────┐  │
│  │  Node 1  │    │  Node 2  │  │
│  │          │    │          │  │
│  │ :30080   │    │ :30080   │  │
│  └────┬─────┘    └────┬─────┘  │
│       │               │         │
│       └───────┬───────┘         │
│               ▼                 │
│          ┌─────────┐            │
│          │ Service │            │
│          │NodePort │            │
│          └─────────┘            │
│               │                 │
│           ┌───┴───┐             │
│           ▼       ▼             │
│        ┌───┐   ┌───┐            │
│        │Pod│   │Pod│            │
│        └───┘   └───┘            │
└─────────────────────────────────┘
        ▲
        │
   ✅ Accessible from outside!
   http://<NodeIP>:30080
```

**Use Cases:**
- Development/testing access
- Simple external access
- When you don't have a LoadBalancer

**Port Range:** 30000-32767 (default)

**Example:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80          # Service port (internal)
    targetPort: 8080  # Pod port
    nodePort: 30080   # External port (optional)
```

**Access:**
```bash
# Get Minikube IP
minikube ip

# Access service
curl http://$(minikube ip):30080

# Or use Minikube helper
minikube service frontend-service
```

#### 3. LoadBalancer

**Purpose:** Expose service externally using cloud provider's load balancer

```
                  ☁ Cloud Provider
                       │
                       ▼
                ┌──────────────┐
                │ Load Balancer│  ◄── External IP: 34.123.45.67
                │   (AWS ELB,  │
                │   GCP LB)    │
                └──────────────┘
                       │
┌──────────────────────┼─────────────────────┐
│         Kubernetes Cluster                 │
│                      │                     │
│                      ▼                     │
│                 ┌─────────┐                │
│                 │ Service │                │
│                 │LoadBalan│                │
│                 └─────────┘                │
│                      │                     │
│              ┌───────┼───────┐            │
│              ▼       ▼       ▼            │
│           ┌───┐   ┌───┐   ┌───┐          │
│           │Pod│   │Pod│   │Pod│          │
│           └───┘   └───┘   └───┘          │
└────────────────────────────────────────────┘
        ▲
        │
   ✅ Accessible from internet!
   http://34.123.45.67
```

**Use Cases:**
- Production applications
- Cloud environments (AWS, GCP, Azure)
- Need real external IP

**Example:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

**Note:** On Minikube, use `minikube tunnel` to simulate LoadBalancer.

#### 4. ExternalName

**Purpose:** Map a Service to an external DNS name (CNAME record)

```
┌─────────────────────────────────┐
│         Kubernetes Cluster      │
│                                 │
│  ┌──────┐                       │
│  │ Pod  │                       │
│  │      │                       │
│  └───┬──┘                       │
│      │                          │
│      │ Connects to:             │
│      │ "database-service"       │
│      │                          │
│      ▼                          │
│  ┌──────────────────┐           │
│  │ ExternalName     │           │
│  │ Service          │           │
│  └──────────────────┘           │
│           │                     │
└───────────┼─────────────────────┘
            │
            ▼
    External DNS resolves to:
    mysql.prod.example.com
```

**Use Cases:**
- Connect to external databases
- Gradually migrate services
- Reference external APIs

**Example:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-database
spec:
  type: ExternalName
  externalName: mysql.prod.example.com
```

**Important Limitation:** ExternalName doesn't work well with HTTPS due to SSL certificate issues. See troubleshooting section for details.

### Service Discovery

Kubernetes provides automatic service discovery:

#### Method 1: DNS (Recommended)

```
Service Name: my-service
Namespace: default

Full DNS Name:
my-service.default.svc.cluster.local

Short form (same namespace):
my-service
```

**Example:**
```bash
# From any pod in the same namespace
curl http://backend-service

# From pod in different namespace
curl http://backend-service.production.svc.cluster.local
```

#### Method 2: Environment Variables (Legacy)

When a Pod starts, it gets env vars for all Services:
```bash
MY_SERVICE_SERVICE_HOST=10.96.0.10
MY_SERVICE_SERVICE_PORT=80
```

**Recommendation:** Use DNS, not environment variables.

### How Services Route Traffic

Services use **kube-proxy** to route traffic:

```
┌────────────────────────────────────┐
│              Node                  │
│                                    │
│  ┌──────────┐    ┌─────────────┐  │
│  │kube-proxy│───►│ iptables    │  │
│  │          │    │ rules       │  │
│  └──────────┘    └─────────────┘  │
│                                    │
│  Traffic to Service IP             │
│         ↓                          │
│  iptables randomly selects         │
│  Pod IP (load balancing)           │
│         ↓                          │
│  ┌────┐  ┌────┐  ┌────┐          │
│  │Pod │  │Pod │  │Pod │          │
│  └────┘  └────┘  └────┘          │
└────────────────────────────────────┘
```

---

## Practical Labs

### Lab 1: The Problem - Pod IPs Change

```bash
# Create a deployment
kubectl create deployment nginx --image=nginx --replicas=3

# Get Pod IPs
kubectl get pods -o wide
# Note the IPs: 10.244.0.5, 10.244.0.6, 10.244.0.7

# Delete one pod
kubectl delete pod <pod-name>

# Check IPs again
kubectl get pods -o wide
# The new pod has a DIFFERENT IP! 💥
```

### Lab 2: Create ClusterIP Service

**nginx-deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.26
        ports:
        - containerPort: 80
```

**nginx-service.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx  # Must match Pod labels!
  ports:
  - protocol: TCP
    port: 80        # Service port
    targetPort: 80  # Pod port
```

```bash
# Create Deployment and Service
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml

# View Service
kubectl get svc nginx-service

# Output:
# NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# nginx-service   ClusterIP   10.96.0.10      <none>        80/TCP    10s

# Describe Service
kubectl describe svc nginx-service

# Shows:
# - Cluster IP: 10.96.0.10 (stable!)
# - Endpoints: 10.244.0.5:80,10.244.0.6:80,10.244.0.7:80
```

### Lab 3: Test ClusterIP Service

```bash
# Create a test pod
kubectl run test-pod --image=busybox --rm -it --restart=Never -- sh

# Inside the test pod:
wget -qO- http://nginx-service
# You should see nginx welcome page!

# Try multiple times - load balancing!
for i in 1 2 3 4 5; do
  wget -qO- http://nginx-service | grep title
done

# Use full DNS name
wget -qO- http://nginx-service.default.svc.cluster.local

exit
```

### Lab 4: View Endpoints

```bash
# View endpoints
kubectl get endpoints nginx-service

# Output:
# NAME            ENDPOINTS                                         AGE
# nginx-service   10.244.0.5:80,10.244.0.6:80,10.244.0.7:80        5m

# Scale deployment
kubectl scale deployment nginx-deployment --replicas=5

# Endpoints automatically updated!
kubectl get endpoints nginx-service
# Now shows 5 IP addresses

# Scale down
kubectl scale deployment nginx-deployment --replicas=2
kubectl get endpoints nginx-service
# Now shows 2 IP addresses
```

### Lab 5: Create NodePort Service

**nginx-nodeport.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80          # Internal cluster port
    targetPort: 80    # Pod port
    nodePort: 30080   # External port (30000-32767)
```

```bash
# Create NodePort service
kubectl apply -f nginx-nodeport.yaml

# View service
kubectl get svc nginx-nodeport

# Output:
# NAME             TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# nginx-nodeport   NodePort   10.96.0.20     <none>        80:30080/TCP   10s

# Get Minikube IP
minikube ip
# Example: 192.168.49.2

# Access from browser
# http://192.168.49.2:30080

# Or use Minikube helper (opens in browser)
minikube service nginx-nodeport

# Or curl it
curl http://$(minikube ip):30080

# if QEMU networking error do this, docker must be installed to do this
minikube stop
minikube delete
minikube start --driver=docker

kubectl run test-pod --image=busybox --rm -it --restart=Never -- sh

# Inside the test pod:
wget -qO- http://$(minikube ip):30080



```

### Lab 6: LoadBalancer Service (Minikube)

**nginx-loadbalancer.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

```bash
# Create LoadBalancer service
kubectl apply -f nginx-loadbalancer.yaml

# Check status
kubectl get svc nginx-loadbalancer
# EXTERNAL-IP shows <pending> on Minikube

# Open a NEW terminal and run:
minikube tunnel
# Keep this running!

# In original terminal, check again:
kubectl get svc nginx-loadbalancer
# Now you'll see an EXTERNAL-IP!

# Access it
curl http://<EXTERNAL-IP>

# Stop tunnel with Ctrl+C in the tunnel terminal
```

### Lab 7: Service with Multiple Ports

**multi-port-service.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-port-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: multi-port
  template:
    metadata:
      labels:
        app: multi-port
    spec:
      containers:
      - name: app
        image: nginx:1.26
        ports:
        - containerPort: 80
          name: http
        - containerPort: 443
          name: https
---
apiVersion: v1
kind: Service
metadata:
  name: multi-port-service
spec:
  selector:
    app: multi-port
  ports:
  - name: http      # Name is REQUIRED for multiple ports
    protocol: TCP
    port: 80
    targetPort: 80
  - name: https
    protocol: TCP
    port: 443
    targetPort: 443
```

```bash
kubectl apply -f multi-port-service.yaml
kubectl describe svc multi-port-service
```

### Lab 8: Service Discovery with DNS

**backend-deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: hashicorp/http-echo
        args:
        - "-text=Hello from Backend!"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 5678
```

**frontend-deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: busybox
        command: ["sh", "-c", "while true; do sleep 3600; done"]
```

```bash
# Deploy backend with service
kubectl apply -f backend-deployment.yaml

# Deploy frontend
kubectl apply -f frontend-deployment.yaml

# Exec into frontend pod
kubectl exec -it <frontend-pod-name> -- sh

# Inside the pod:

# Short name (same namespace)
wget -qO- http://backend-service

# Full DNS name
wget -qO- http://backend-service.default.svc.cluster.local

# DNS resolution
nslookup backend-service
# Shows: backend-service.default.svc.cluster.local → 10.96.0.X

# Test load balancing
for i in 1 2 3 4 5; do
  wget -qO- http://backend-service
  echo ""
done

exit
```

### Lab 9: Headless Service

**headless-service.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-service
spec:
  clusterIP: None  # Makes it headless!
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f headless-service.yaml

# Check service
kubectl get svc headless-service

# Output:
# NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
# headless-service   ClusterIP   None         <none>        80/TCP    10s

# DNS returns Pod IPs directly (not Service IP)
kubectl run test --image=busybox --restart=Never --command -- sh -c "sleep 3600"
kubectl exec -it test -- nslookup headless-service

# Shows multiple A records, one for each Pod!
```

---

## Common Commands

### Service Management

```bash
# Create Service imperatively
kubectl create service clusterip my-service --tcp=80:8080
kubectl create service nodeport my-service --tcp=80:8080 --node-port=30080
kubectl expose deployment my-deployment --port=80 --target-port=8080

# View Services
kubectl get services
kubectl get svc
kubectl get svc -o wide
kubectl describe svc <name>

# View Endpoints
kubectl get endpoints
kubectl get ep <service-name>
kubectl describe ep <service-name>

# Test Service from within cluster
kubectl run test --image=busybox --rm -it -- wget -qO- http://<service-name>

# Get Service URL (Minikube)
minikube service <service-name> --url
minikube service <service-name>  # Opens in browser

# Port forward (for testing)
kubectl port-forward svc/<service-name> 8080:80
# Access: http://localhost:8080

# Delete Service
kubectl delete service <name>
kubectl delete svc <name>
kubectl delete -f service.yaml

# Edit Service
kubectl edit svc <name>

# Export YAML
kubectl get svc <name> -o yaml > service-backup.yaml
```

### Debugging Commands

```bash
# Check Service
kubectl get svc <service-name>

# Check Endpoints
kubectl get endpoints <service-name>

# Test DNS resolution
kubectl run test --image=busybox --restart=Never --command -- sh -c "sleep 3600"
kubectl exec -it test -- nslookup <service-name>

# Test Service connectivity
kubectl run test --image=curlimages/curl --rm -it -- curl http://<service-name>

# Check Pod labels
kubectl get pods --show-labels

# Check Service selector
kubectl describe svc <service-name>

# View all in one
kubectl get deploy,svc,ep,pods

# Force delete pod
kubectl delete pod test --grace-period=0 --force

```

---

## Key Takeaways

### Core Concepts

✅ **Services provide stable networking** - IPs and DNS don't change  
✅ **Services use selectors** to find Pods with matching labels  
✅ **Services load balance** automatically across Pod replicas  
✅ **ClusterIP** for internal communication (default)  
✅ **NodePort** for external access in dev/test  
✅ **LoadBalancer** for production external access (cloud)  
✅ **DNS-based discovery** is the recommended approach  
✅ **Endpoints track Pod IPs** automatically  

### Service Type Decision Tree

```
Do you need external access?
├─ NO → Use ClusterIP
│      (backend services, databases)
│
└─ YES → Is this production?
   ├─ NO → Use NodePort
   │      (development, testing)
   │
   └─ YES → On cloud provider?
      ├─ YES → Use LoadBalancer
      │       (AWS, GCP, Azure)
      │
      └─ NO → Use NodePort or Ingress
              (on-premises, Minikube)
```

### Best Practices

1. **Always use Services** - never connect to Pods directly
2. **Use ClusterIP by default** - only expose externally when needed
3. **Match selector labels** carefully - typos break everything
4. **Use DNS names** - not environment variables
5. **Name your ports** - especially with multiple ports
6. **Set resource limits** on Pods behind Services
7. **Use readiness probes** - Services only route to ready Pods
8. **Monitor Endpoints** - ensure they match expected Pod count

---

## Homework

### Task 1: Three-Tier Application

Create a complete three-tier application:

**Database Tier:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16
        env:
        - name: POSTGRES_PASSWORD
          value: mysecretpassword
        ports:
        - containerPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

**Backend Tier:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: hashicorp/http-echo
        args:
        - "-text=Backend API Response"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 5678
```

**Frontend Tier:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.26
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30100
```

```bash
kubectl apply -f database-tier.yaml
kubectl apply -f backend-tier.yaml
kubectl apply -f frontend-tier.yaml

# Verify
kubectl get deploy,svc,pods

# Access frontend
minikube service frontend-service
```

### Task 2: Test Service Discovery

```bash
# Exec into frontend pod
kubectl exec -it <frontend-pod> -- sh

# Test DNS resolution
nslookup backend-service
nslookup postgres-service

# Test connectivity
curl http://backend-service
# Should get: "Backend API Response"

# Full DNS names
curl http://backend-service.default.svc.cluster.local
```

### Task 3: Load Balancing Test

```bash
# Scale backend to 5 replicas
kubectl scale deployment backend --replicas=5

# From frontend pod, make 20 requests
kubectl exec -it <frontend-pod> -- sh
for i in $(seq 1 20); do
  curl http://backend-service
  echo ""
done

# Check which backend pods received requests
kubectl logs <backend-pod-1>
kubectl logs <backend-pod-2>
# etc.
```

### Task 4: Service Types Comparison

Create the same deployment with all three service types and document:

**Questions:**
1. Which service types are accessible from inside the cluster?
2. Which service types are accessible from your browser?
3. What are the port numbers for each type?
4. Which type would you use for production?

### Task 5: Multi-Port Service

Create a service that exposes:
- HTTP on port 80
- HTTPS on port 443
- Admin on port 8080

Use named ports in both Deployment and Service.

### Task 6: Headless Service Investigation

1. Create a headless service (clusterIP: None)
2. Query DNS: `nslookup <headless-service>`
3. Compare with normal ClusterIP service
4. When would you use headless services?

**Answer:** Headless services are used with StatefulSets when you need direct access to individual Pods (e.g., for databases in a cluster).

---

## Troubleshooting

### Service Not Working

```bash
# 1. Check Service exists
kubectl get svc <service-name>

# 2. Check Endpoints exist
kubectl get endpoints <service-name>
# If no endpoints, selector doesn't match Pods!

# 3. Check Pod labels
kubectl get pods --show-labels
# Labels must match Service selector

# 4. Check Service selector
kubectl describe svc <service-name>
# Look at "Selector" field

# 5. Test from within cluster
kubectl run test --image=busybox --rm -it -- wget -qO- http://<service-name>

# 6. Check Pods are ready
kubectl get pods
# STATUS should be Running, READY should be 1/1

# 7. Check Pod logs
kubectl logs <pod-name>
```

### Common Issues

**No Endpoints:**
- **Cause:** Selector doesn't match Pod labels
- **Solution:** Fix labels or selector
```bash
# Check Pod labels
kubectl get pods --show-labels

# Check Service selector
kubectl describe svc <service-name>

# They must match!
```

**Connection Refused:**
- **Cause:** Wrong targetPort in Service or container not listening
- **Solution:** Check container port matches Service targetPort
```bash
# Check container port
kubectl describe pod <pod-name>

# Check Service targetPort
kubectl describe svc <service-name>
```

**DNS Not Resolving:**
- **Cause:** CoreDNS not running
- **Solution:** Check CoreDNS pods
```bash
kubectl get pods -n kube-system | grep coredns
# Should show running CoreDNS pods

# If not running:
kubectl logs -n kube-system <coredns-pod>
```

**NodePort Not Accessible:**
- **Cause:** Wrong node IP or firewall blocking
- **Solution:** Check Minikube IP and firewall
```bash
# Get correct IP
minikube ip

# Test connection
curl http://$(minikube ip):<nodePort>

# Check Service
kubectl get svc <service-name>
```

**LoadBalancer Pending (Minikube):**
- **Cause:** Minikube doesn't provide LoadBalancer by default
- **Solution:** Use minikube tunnel
```bash
# In a separate terminal:
minikube tunnel

# Keep it running
```

### ExternalName Service SSL Issues

**Problem:** ExternalName services don't work with HTTPS

```bash
# This FAILS:
kubectl run test --rm -it --image=curlimages/curl -- curl https://external-api
# Error: SSL certificate doesn't match

# This WORKS:
kubectl run test --rm -it --image=curlimages/curl -- curl https://api.github.com
```

**Why:**
- ExternalName creates DNS CNAME only
- HTTP Host header still uses service name
- SSL certificate doesn't match service name

**Solution:**
- Use actual domain for HTTPS: `curl https://api.github.com`
- Use ExternalName only for non-HTTPS services
- For HTTPS APIs, use a proxy pod (nginx/envoy)

---

## Clean Up

```bash
# Delete all resources from this lesson
kubectl delete deployment nginx-deployment
kubectl delete deployment backend
kubectl delete deployment frontend
kubectl delete deployment postgres
kubectl delete deployment multi-port-app

kubectl delete service nginx-service
kubectl delete service nginx-nodeport
kubectl delete service nginx-loadbalancer
kubectl delete service backend-service
kubectl delete service frontend-service
kubectl delete service postgres-service
kubectl delete service multi-port-service
kubectl delete service headless-service

# Verify cleanup
kubectl get deploy,svc,pods
```

---

## What's Next?

**You now know:**
- ✅ Why Services are essential for networking
- ✅ The four Service types and when to use each
- ✅ How to expose applications internally and externally
- ✅ Service discovery with DNS