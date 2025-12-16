# Kubernetes Lesson 5: Ingress - Advanced HTTP/HTTPS Routing

## Table of Contents
- [Overview](#overview)
- [Theory](#theory)
  - [The Problem with Services](#the-problem-with-services)
  - [What is Ingress?](#what-is-ingress)
  - [Ingress Architecture](#ingress-architecture)
  - [Ingress vs Services](#ingress-vs-services)
  - [Routing Types](#routing-types)
- [Practical Labs](#practical-labs)
- [Common Commands](#common-commands)
- [Key Takeaways](#key-takeaways)
- [Homework](#homework)
- [Troubleshooting](#troubleshooting)

---

## Overview

This lesson covers Kubernetes Ingress, which provides HTTP/HTTPS routing to services with advanced features like path-based routing, host-based routing, and SSL/TLS termination.

**Learning Objectives:**
- Understand the limitations of Services for web applications
- Learn what Ingress is and why it's needed
- Master path-based and host-based routing
- Configure TLS/HTTPS with certificates
- Use annotations for advanced features
- Troubleshoot common Ingress issues

---

## Theory

### The Problem with Services

**Scenario:** You have 3 web applications

Using NodePort for each:
```bash
app1-service   NodePort   10.96.0.10   30001/TCP
app2-service   NodePort   10.96.0.11   30002/TCP
app3-service   NodePort   10.96.0.12   30003/TCP
```

**Users must access:**
```bash
http://mysite.com:30001  # App 1
http://mysite.com:30002  # App 2
http://mysite.com:30003  # App 3
```

**Problems:**
- ❌ Different ports for each app (ugly URLs)
- ❌ No SSL/TLS termination
- ❌ No path-based routing (can't use `/app1`, `/app2`)
- ❌ No host-based routing (can't use `app1.mysite.com`)
- ❌ Need one LoadBalancer per service (expensive in cloud!)
- ❌ No central place for security rules

**What we want:**
```bash
https://mysite.com/app1     → App 1 Service
https://mysite.com/app2     → App 2 Service
https://mysite.com/api      → API Service

# OR subdomain-based:
https://app1.mysite.com     → App 1 Service
https://app2.mysite.com     → App 2 Service
https://api.mysite.com      → API Service
```

### What is Ingress?

An **Ingress** is a Kubernetes API object that manages external HTTP/HTTPS access to services in a cluster.

**What Ingress Provides:**
- ✅ **Single entry point** for multiple services
- ✅ **Path-based routing** (e.g., `/app1`, `/app2`)
- ✅ **Host-based routing** (e.g., `app1.com`, `app2.com`)
- ✅ **SSL/TLS termination** (handles HTTPS)
- ✅ **Load balancing** across pods
- ✅ **Name-based virtual hosting**
- ✅ **Cost-effective** (one LoadBalancer for many services)

### Ingress Architecture

```
                    Internet
                       │
                       ▼
              ┌─────────────────┐
              │  LoadBalancer   │  (Single external IP)
              │  or NodePort    │
              └─────────────────┘
                       │
                       ▼
              ┌─────────────────┐
              │ Ingress         │  (Routes based on rules)
              │ Controller      │
              │ (nginx/traefik) │
              └─────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
   ┌─────────┐   ┌─────────┐   ┌─────────┐
   │Service1 │   │Service2 │   │Service3 │
   │ClusterIP│   │ClusterIP│   │ClusterIP│
   └─────────┘   └─────────┘   └─────────┘
        │              │              │
    ┌───┴───┐     ┌───┴───┐     ┌───┴───┐
    │ Pods  │     │ Pods  │     │ Pods  │
    └───────┘     └───────┘     └───────┘
```

### Ingress Components

**1. Ingress Resource** - YAML configuration defining routing rules  
**2. Ingress Controller** - Software that implements the rules (nginx, Traefik, HAProxy, etc.)

```
Ingress Resource (config) + Ingress Controller (implementation) = Working Ingress
```

**Important:** Ingress resources do nothing without an Ingress Controller!

### Ingress vs Services

| Feature | Service (LoadBalancer) | Ingress |
|---------|----------------------|---------|
| External access | ✅ | ✅ |
| Path-based routing | ❌ | ✅ |
| Host-based routing | ❌ | ✅ |
| SSL/TLS termination | ❌ | ✅ |
| Multiple services | Need multiple LBs 💰 | Single entry point ✅ |
| HTTP/HTTPS only | ❌ (any protocol) | ✅ (HTTP/HTTPS only) |
| Cost (cloud) | High (per LB) | Low (single LB) |

### Routing Types

#### 1. Path-Based Routing (Simple Fanout)

```
myapp.com/foo    → service1:80
myapp.com/bar    → service2:80
myapp.com/api    → service3:80
```

```
┌─────────────────────────────┐
│    http://myapp.com         │
└─────────────────────────────┘
              │
    ┌─────────┼─────────┐
    │         │         │
    ▼         ▼         ▼
  /foo      /bar      /api
    │         │         │
    ▼         ▼         ▼
Service1  Service2  Service3
```

#### 2. Host-Based Routing (Virtual Hosting)

```
foo.myapp.com    → service1:80
bar.myapp.com    → service2:80
api.myapp.com    → service3:80
```

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│foo.myapp.com │  │bar.myapp.com │  │api.myapp.com │
└──────────────┘  └──────────────┘  └──────────────┘
       │                 │                 │
       ▼                 ▼                 ▼
   Service1          Service2          Service3
```

#### 3. TLS/HTTPS

```
https://myapp.com → terminates SSL → routes to services
```

### Basic Ingress Structure

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
```

**Key Fields:**
- **rules**: List of routing rules
- **host**: Domain name (optional, matches all if omitted)
- **paths**: URL paths to match
- **pathType**: How to match (Prefix, Exact, ImplementationSpecific)
- **backend**: Target service

### Path Types

| Path Type | Behavior | Example |
|-----------|----------|---------|
| **Prefix** | Matches path prefix | `/foo` matches `/foo`, `/foo/`, `/foo/bar` |
| **Exact** | Exact string match | `/foo` matches ONLY `/foo` |
| **ImplementationSpecific** | Controller-specific | Depends on controller |

### Popular Ingress Controllers

1. **NGINX Ingress Controller** (most popular)
2. **Traefik** (modern, easy)
3. **HAProxy Ingress**
4. **Kong Ingress**
5. **Istio Ingress Gateway**
6. **AWS ALB Ingress** (AWS specific)
7. **GCE Ingress** (GCP specific)

---

## Practical Labs

### Lab 0: Enable Ingress Controller (Minikube)

```bash
# Enable NGINX Ingress addon
minikube addons enable ingress

# Verify it's running
kubectl get pods -n ingress-nginx

# Expected output:
# NAME                                        READY   STATUS
# ingress-nginx-controller-xxxxx              1/1     Running

# Wait for controller to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

### Lab 1: Simple Single Service Ingress

**app-deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: hashicorp/http-echo
        args:
        - "-text=Hello from Web App!"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 5678
```

**simple-ingress.yaml:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

```bash
# Deploy
kubectl apply -f app-deployment.yaml
kubectl apply -f simple-ingress.yaml

# View Ingress
kubectl get ingress
# NAME             CLASS   HOSTS   ADDRESS        PORTS   AGE
# simple-ingress   nginx   *       192.168.49.2   80      10s

# Test it
curl http://$(minikube ip)

# if Docker do port forward and check using localhost
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80
curl http://localhost:8080

# Output: Hello from Web App!
```

### Lab 2: Path-Based Routing (Fanout)

**multi-app-deployment.yaml:**
```yaml
# App 1
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1
        image: hashicorp/http-echo
        args:
        - "-text=Response from App 1"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: app1-service
spec:
  selector:
    app: app1
  ports:
  - port: 80
    targetPort: 5678
---
# App 2
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - name: app2
        image: hashicorp/http-echo
        args:
        - "-text=Response from App 2"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: app2-service
spec:
  selector:
    app: app2
  ports:
  - port: 80
    targetPort: 5678
---
# API Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo
        args:
        - "-text=API Response"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 5678
```

**fanout-ingress.yaml:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fanout-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

```bash
# Deploy
kubectl apply -f multi-app-deployment.yaml
kubectl apply -f fanout-ingress.yaml

# Test different paths
curl http://$(minikube ip)/app1
# Output: Response from App 1

curl http://$(minikube ip)/app2
# Output: Response from App 2

curl http://$(minikube ip)/api
# Output: API Response
```

**Understanding rewrite-target:**
```
User request: http://myapp.com/app1/some/path
                               ↓
Without rewrite:
  Backend receives: GET /app1/some/path  ← App might not handle /app1 prefix

With nginx.ingress.kubernetes.io/rewrite-target: /
  Backend receives: GET /some/path       ← Clean path
```

### Lab 3: Host-Based Routing (Virtual Hosting)

**host-based-ingress.yaml:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
spec:
  rules:
  - host: app1.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  - host: app2.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
  - host: api.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

```bash
# Create Ingress
kubectl apply -f host-based-ingress.yaml

# Get Minikube IP
minikube ip
# Example: 192.168.49.2

# Option 1: Edit /etc/hosts (Linux/Mac)
sudo nano /etc/hosts
# Add these lines (replace with your Minikube IP):
# 192.168.49.2 app1.local
# 192.168.49.2 app2.local
# 192.168.49.2 api.local

# Then test:
curl http://app1.local
curl http://app2.local
curl http://api.local

# Option 2: Use Host header (no /etc/hosts edit)
curl -H "Host: app1.local" http://$(minikube ip)
# Output: Response from App 1

curl -H "Host: app2.local" http://$(minikube ip)
# Output: Response from App 2

curl -H "Host: api.local" http://$(minikube ip)
# Output: API Response
```

### Lab 4: Combined Host and Path Routing

**combined-ingress.yaml:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: combined-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
  - host: api.myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

```bash
kubectl apply -f combined-ingress.yaml

# Test
curl -H "Host: myapp.local" http://$(minikube ip)/v1
# Output: Response from App 1

curl -H "Host: myapp.local" http://$(minikube ip)/v2
# Output: Response from App 2

curl -H "Host: api.myapp.local" http://$(minikube ip)
# Output: API Response
```

### Lab 5: Default Backend

**default-backend.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: default-backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: default
  template:
    metadata:
      labels:
        app: default
    spec:
      containers:
      - name: default
        image: hashicorp/http-echo
        args:
        - "-text=404 - Page Not Found"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: default-backend-service
spec:
  selector:
    app: default
  ports:
  - port: 80
    targetPort: 5678
```

**ingress-with-default.yaml:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-with-default
spec:
  defaultBackend:
    service:
      name: default-backend-service
      port:
        number: 80
  rules:
  - host: app1.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
```

```bash
kubectl apply -f default-backend.yaml
kubectl apply -f ingress-with-default.yaml

# Test valid route
curl -H "Host: app1.local" http://$(minikube ip)
# Output: Response from App 1

# Test non-existent host
curl -H "Host: nonexistent.local" http://$(minikube ip)
# Output: 404 - Page Not Found
```

### Lab 6: TLS/HTTPS Ingress

**Step 1: Create self-signed certificate**
```bash
# Generate certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=myapp.local/O=myapp"

# Create Kubernetes secret
kubectl create secret tls myapp-tls \
  --cert=tls.crt \
  --key=tls.key

# Verify secret
kubectl get secret myapp-tls
```

**tls-ingress.yaml:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - myapp.local
    secretName: myapp-tls
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
```

```bash
kubectl apply -f tls-ingress.yaml

# Test HTTPS (-k to ignore self-signed cert warning)
curl -k -H "Host: myapp.local" https://$(minikube ip)
# Output: Response from App 1

# Check certificate details
curl -k -v -H "Host: myapp.local" https://$(minikube ip) 2>&1 | grep "subject:"

```

### Lab 7: Path Type Comparison

**pathtype-comparison.yaml:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pathtype-demo
spec:
  rules:
  - http:
      paths:
      # Prefix matching
      - path: /prefix
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
      # Exact matching
      - path: /exact
        pathType: Exact
        backend:
          service:
            name: app2-service
            port:
              number: 80
```

```bash
kubectl apply -f pathtype-comparison.yaml

# Prefix matches /prefix, /prefix/, /prefix/anything
curl http://$(minikube ip)/prefix          # ✅ Matches
curl http://$(minikube ip)/prefix/         # ✅ Matches
curl http://$(minikube ip)/prefix/test     # ✅ Matches

# Exact only matches /exact exactly
curl http://$(minikube ip)/exact           # ✅ Matches
curl http://$(minikube ip)/exact/          # ❌ Does NOT match (404)
curl http://$(minikube ip)/exact/test      # ❌ Does NOT match (404)
```

### Lab 8: Common Annotations

**annotations-ingress.yaml:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: annotations-demo
  annotations:
    # Rewrite target URL
    nginx.ingress.kubernetes.io/rewrite-target: /
    
    # SSL redirect (force HTTPS)
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    
    # Connection timeout (seconds)
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    
    # Read timeout (seconds)
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    
    # Request body size limit
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    
    # Rate limiting (requests per second)
    nginx.ingress.kubernetes.io/limit-rps: "10"
    
    # Custom response headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Custom-Header: MyValue";
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
```

```bash
kubectl apply -f annotations-ingress.yaml

# Test and check headers
curl -v http://$(minikube ip) 2>&1 | grep "X-Custom-Header"
```

### Lab 9: Wildcard Host

**wildcard-ingress.yaml:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wildcard-ingress
spec:
  rules:
  - host: "*.myapp.local"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
```

```bash
kubectl apply -f wildcard-ingress.yaml

# All subdomains work
curl -H "Host: anything.myapp.local" http://$(minikube ip)
curl -H "Host: test.myapp.local" http://$(minikube ip)
curl -H "Host: dev.myapp.local" http://$(minikube ip)
# All route to app1-service
```

---

## Common Commands

### View Ingress Resources

```bash
# List Ingress
kubectl get ingress
kubectl get ing  # Short form
kubectl get ingress -o wide

# Describe Ingress
kubectl describe ingress <name>

# Get Ingress YAML
kubectl get ingress <name> -o yaml

# Get Ingress in all namespaces
kubectl get ingress --all-namespaces
kubectl get ingress -A  # Short form
```

### Ingress Controller

```bash
# Check controller status
kubectl get pods -n ingress-nginx

# View controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Follow controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx -f

# Describe controller pod
kubectl describe pod -n ingress-nginx <controller-pod-name>
```

### Create/Update Ingress

```bash
# Create from file
kubectl apply -f ingress.yaml

# Create from stdin
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: test-service
            port:
              number: 80
EOF

# Edit Ingress
kubectl edit ingress <name>

# Delete Ingress
kubectl delete ingress <name>
kubectl delete -f ingress.yaml
```

### Testing Ingress

```bash
# Test with curl
curl http://$(minikube ip)

# Test with Host header
curl -H "Host: myapp.local" http://$(minikube ip)

# Test HTTPS
curl -k https://$(minikube ip)

# Verbose output
curl -v http://$(minikube ip)

# Follow redirects
curl -L http://$(minikube ip)

```

### TLS/Secrets

```bash
# Create TLS secret
kubectl create secret tls <secret-name> \
  --cert=path/to/cert.crt \
  --key=path/to/key.key

# View secrets
kubectl get secrets

# Describe secret
kubectl describe secret <secret-name>

# Delete secret
kubectl delete secret <secret-name>
```

### Debugging

```bash
# Check Ingress events
kubectl get events --field-selector involvedObject.name=<ingress-name>

# Check all events
kubectl get events --sort-by='.lastTimestamp'

# Validate backend service
kubectl get svc <service-name>

# Check endpoints
kubectl get endpoints <service-name>

# Check pods
kubectl get pods -l app=<app-label>

# Test service directly
kubectl port-forward svc/<service-name> 8080:80
# Then: curl http://localhost:8080
```

---

## Key Takeaways

### Core Concepts

✅ **Ingress manages HTTP/HTTPS access** to multiple services  
✅ **Ingress Controller required** - nginx, traefik, etc.  
✅ **Path-based routing** - `/app1`, `/app2`, `/api`  
✅ **Host-based routing** - `app1.com`, `app2.com`  
✅ **TLS termination** - handle HTTPS at Ingress level  
✅ **Single entry point** - one LoadBalancer for all services  
✅ **Cost-effective** - reduces cloud costs significantly  
✅ **Annotations for features** - timeouts, redirects, rate limits  

### Decision Tree: Service vs Ingress

```
Need external access?
├─ HTTP/HTTPS only?
│  ├─ YES → Multiple services?
│  │  ├─ YES → Use Ingress ✅
│  │  └─ NO → Use Service or Ingress
│  └─ NO (TCP/UDP/gRPC) → Use Service
└─ Internal only? → Use ClusterIP Service
```

### When to Use Ingress

**✅ Use Ingress when:**
- Multiple HTTP/HTTPS services to expose
- Need path-based routing (`/api`, `/web`)
- Need host-based routing (`api.example.com`)
- Need SSL/TLS termination
- Want to save costs (cloud LoadBalancers)
- Production web applications

**❌ Don't use Ingress when:**
- Non-HTTP protocols (TCP, UDP, custom)
- Single simple service (Service is easier)
- No routing requirements
- Learning basics (start with Services)

### Best Practices

1. **Always use ClusterIP services** behind Ingress (not NodePort/LoadBalancer)
2. **Use meaningful hostnames** for organization
3. **Enable TLS in production** - use cert-manager for automation
4. **Set resource limits** on backend pods
5. **Use annotations wisely** - understand what they do
6. **Monitor Ingress Controller** - logs, metrics, health
7. **Implement default backend** for custom 404 pages
8. **Test with curl first** before DNS changes
9. **Use path prefixes carefully** to avoid conflicts
10. **Document routing rules** - they get complex quickly
11. **Version your Ingress configs** - treat as code
12. **Use namespaces** to organize Ingress resources

### Common Annotations (NGINX)

| Annotation | Purpose | Example |
|------------|---------|---------|
| `rewrite-target` | Rewrite URL path | `/` |
| `ssl-redirect` | Force HTTPS | `"true"` |
| `proxy-connect-timeout` | Connection timeout | `"30"` |
| `proxy-read-timeout` | Read timeout | `"60"` |
| `proxy-body-size` | Max request size | `"10m"` |
| `limit-rps` | Rate limiting | `"10"` |
| `auth-type` | Authentication | `"basic"` |
| `whitelist-source-range` | IP whitelist | `"10.0.0.0/8"` |
| `canary` | Canary deployment | `"true"` |
| `canary-weight` | Traffic split | `"10"` |

---

## Homework

### Task 1: E-Commerce Application

Create a complete e-commerce Ingress with:

**Services:**
- Frontend (website) - `/`
- User API - `/api/users`
- Product API - `/api/products`
- Cart API - `/api/cart`
- Payment API - `/api/payment`
- Admin dashboard - `/admin`

**Requirements:**
- All on same host (`shop.local`)
- Enable HTTPS with TLS
- Add default 404 backend
- Use path-based routing

**Starter Template:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - shop.local
    secretName: shop-tls
  defaultBackend:
    service:
      name: default-backend
      port:
        number: 80
  rules:
  - host: shop.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      - path: /api/users
        pathType: Prefix
        backend:
          service:
            name: user-api-service
            port:
              number: 80
      # Add more paths...
```

### Task 2: Multi-Environment Routing

Create separate environments with host-based routing:

**Requirements:**
- `dev.myapp.local` → dev-service
- `staging.myapp.local` → staging-service
- `prod.myapp.local` → prod-service

**Steps:**
1. Create 3 deployments (use different text for each)
2. Create 3 services
3. Create Ingress with host-based rules
4. Test each environment

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-env-ingress
spec:
  rules:
  - host: dev.myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: dev-service
            port:
              number: 80
  # Add staging and prod...
```

### Task 3: API Versioning

Create API version routing:

**Requirements:**
- `/api/v1/...` → api-v1-service
- `/api/v2/...` → api-v2-service
- `/api/v3/...` → api-v3-service

**Test:**
```bash
curl http://$(minikube ip)/api/v1/users
curl http://$(minikube ip)/api/v2/users
curl http://$(minikube ip)/api/v3/users
```

### Task 4: Custom Error Pages

**Requirements:**
1. Create custom 404 service (returns "Custom 404 Page")
2. Create custom 503 service (returns "Under Maintenance")
3. Configure Ingress to use them
4. Test by accessing non-existent paths

**Hint:** Use `default-backend` annotation

### Task 5: Rate Limiting

Create an Ingress with rate limiting:

**Requirements:**
- Limit to 5 requests per second
- Return 429 (Too Many Requests) when exceeded

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rate-limited-ingress
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "5"
spec:
  # Your rules here
```

**Test:**
```bash
# Should succeed for first 5, then fail
for i in {1..20}; do 
  curl http://$(minikube ip)
  echo ""
done
```

### Task 6: HTTPS Redirect

**Requirements:**
1. Accept both HTTP and HTTPS
2. Automatically redirect HTTP to HTTPS
3. Use proper TLS certificate

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: https-redirect-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - myapp.local
    secretName: myapp-tls
  # Your rules here
```

**Test:**
```bash
# Should redirect to HTTPS
curl -L http://$(minikube ip)

# Verify redirect
curl -v http://$(minikube ip) 2>&1 | grep "301\|302"
```

### Task 7: Canary Deployment (Advanced)

Split traffic between two versions:
- 90% to stable version
- 10% to canary version

```yaml
# Main Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
spec:
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v1-service
            port:
              number: 80
---
# Canary Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary-ingress
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v2-service
            port:
              number: 80
```

---

## Troubleshooting

### Common Issues and Solutions

#### Issue 1: Ingress Not Working

**Symptoms:**
- `kubectl get ingress` shows no ADDRESS
- Cannot access application

**Debug Steps:**
```bash
# 1. Check if Ingress Controller is running
kubectl get pods -n ingress-nginx

# Expected: controller pod in Running state
# If not running:
minikube addons enable ingress

# 2. Check Ingress resource
kubectl describe ingress <ingress-name>
# Look for errors in Events section

# 3. Verify backend service exists
kubectl get svc <service-name>

# 4. Check endpoints exist
kubectl get endpoints <service-name>
# If empty, selector doesn't match pod labels

# 5. Check Ingress Controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=50
```

#### Issue 2: 404 Not Found

**Possible Causes:**
- Wrong path in Ingress
- Path type mismatch (Prefix vs Exact)
- Missing rewrite-target annotation
- Service name incorrect
- Backend service not running

**Debug:**
```bash
# Check Ingress configuration
kubectl describe ingress <ingress-name>

# Verify service and endpoints
kubectl get svc <service-name>
kubectl get endpoints <service-name>

# Check pods are running
kubectl get pods -l app=<your-app>

# Test service directly
kubectl port-forward svc/<service-name> 8080:80
curl http://localhost:8080

# Check Ingress path matching
curl -v http://$(minikube ip)/your-path
```

#### Issue 3: 503 Service Unavailable

**Possible Causes:**
- No healthy pods in backend
- Pods not ready (readiness probe failing)
- Service selector doesn't match pods

**Debug:**
```bash
# Check pod status
kubectl get pods -l app=<your-app>

# Check pod details
kubectl describe pod <pod-name>
# Look for readiness probe failures

# Check service selector
kubectl describe svc <service-name>

# Check if endpoints exist
kubectl get endpoints <service-name>
# Should show pod IPs

# Check pod logs
kubectl logs <pod-name>
```

#### Issue 4: TLS/SSL Issues

**Possible Causes:**
- Certificate doesn't match hostname
- Secret not found
- Secret in wrong namespace
- Certificate expired

**Debug:**
```bash
# Check if secret exists
kubectl get secret <tls-secret-name>

# Describe secret
kubectl describe secret <tls-secret-name>

# Verify certificate
kubectl get secret <tls-secret-name> -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout | grep -A2 "Subject:"

# Check certificate expiry
kubectl get secret <tls-secret-name> -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout | grep "Not After"

# Test HTTPS connection
curl -k -v https://$(minikube ip) 2>&1 | grep "SSL\|TLS"
```

#### Issue 5: Host Not Resolving

**Possible Causes:**
- Host not in /etc/hosts
- DNS not configured
- Typo in hostname

**Solutions:**
```bash
# Option 1: Add to /etc/hosts (for testing)
sudo nano /etc/hosts
# Add: 192.168.49.2 myapp.local

# Option 2: Use Host header with curl
curl -H "Host: myapp.local" http://$(minikube ip)

# Option 3: Use real DNS (production)
# Configure DNS A record pointing to LoadBalancer IP
```

#### Issue 6: Path Rewriting Issues

**Symptoms:**
- Backend receives wrong path
- Application returns 404 even though Ingress routes correctly

**Debug:**
```bash
# Check if rewrite-target is set
kubectl get ingress <name> -o yaml | grep rewrite-target

# Test with rewrite-target
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /

# Test without rewrite-target (if app handles full path)
# Remove the annotation

# Check what path backend receives
kubectl logs <pod-name>
```

#### Issue 7: Annotations Not Working

**Possible Causes:**
- Typo in annotation name
- Wrong Ingress Controller (annotation is controller-specific)
- Invalid annotation value

**Debug:**
```bash
# Check annotation syntax
kubectl get ingress <name> -o yaml

# View controller logs for errors
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx | grep -i error

# Verify annotation is controller-specific
# NGINX: nginx.ingress.kubernetes.io/...
# Traefik: traefik.ingress.kubernetes.io/...
```

### Debugging Checklist

```bash
# Quick debugging checklist
□ Ingress Controller running? → kubectl get pods -n ingress-nginx
□ Ingress resource created? → kubectl get ingress
□ Ingress has ADDRESS? → kubectl get ingress
□ Backend service exists? → kubectl get svc <service-name>
□ Service has endpoints? → kubectl get endpoints <service-name>
□ Pods are running? → kubectl get pods
□ Pods are ready? → kubectl get pods (check READY column)
□ Path matches request? → Check pathType and path value
□ Host header correct? → Use curl -H "Host: ..."
□ TLS secret exists? → kubectl get secret <tls-secret>
□ Check controller logs? → kubectl logs -n ingress-nginx ...
```

### Useful Debugging Commands

```bash
# Get full Ingress details
kubectl get ingress <name> -o yaml

# Watch Ingress updates
kubectl get ingress -w

# Test with verbose curl
curl -v http://$(minikube ip)

# Check specific header
curl -H "Host: myapp.local" -v http://$(minikube ip) 2>&1 | grep "HTTP"

# Follow controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx -f

# Check all events
kubectl get events --sort-by='.lastTimestamp'

# Port forward to test backend directly
kubectl port-forward svc/<service-name> 8080:80
curl http://localhost:8080

# Exec into controller pod (advanced)
kubectl exec -it -n ingress-nginx <controller-pod> -- /bin/bash
# Inside: cat /etc/nginx/nginx.conf
```

---

## Clean Up

```bash
# Delete all Ingress resources
kubectl delete ingress --all

# Delete specific Ingress
kubectl delete ingress simple-ingress fanout-ingress host-based-ingress

# Delete deployments
kubectl delete deployment web-app app1 app2 api default-backend

# Delete services
kubectl delete service web-service app1-service app2-service api-service default-backend-service

# Delete TLS secrets
kubectl delete secret myapp-tls shop-tls

# Delete generated certificate files (if created locally)
rm -f tls.key tls.crt

# Disable Ingress addon (optional, if no longer needed)
# minikube addons disable ingress

# Verify cleanup
kubectl get ingress,deploy,svc,pods,secrets
```

---

## Additional Resources

### Official Documentation
- [Kubernetes Ingress Documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [Ingress Controllers List](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)

### NGINX Ingress Annotations
- [Full Annotation Reference](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/)
- [Examples Repository](https://github.com/kubernetes/ingress-nginx/tree/main/docs/examples)

### TLS/Certificate Management
- [cert-manager](https://cert-manager.io/) - Automated certificate management
- [Let's Encrypt](https://letsencrypt.org/) - Free SSL certificates

### Tools
- [k9s](https://k9scli.io/) - Terminal UI for Kubernetes
- [kubectx/kubens](https://github.com/ahmetb/kubectx) - Context and namespace switching
- [stern](https://github.com/stern/stern) - Multi-pod log tailing

---

## Comparison: Service Types vs Ingress

| Feature | ClusterIP | NodePort | LoadBalancer | Ingress |
|---------|-----------|----------|--------------|---------|
| **Scope** | Internal only | External | External | External |
| **Protocol** | Any | Any | Any | HTTP/HTTPS only |
| **Port** | Any | 30000-32767 | 80/443/any | 80/443 |
| **Path routing** | ❌ | ❌ | ❌ | ✅ |
| **Host routing** | ❌ | ❌ | ❌ | ✅ |
| **TLS termination** | ❌ | ❌ | ❌ | ✅ |
| **Cost (cloud)** | Free | Free | $$ per service | $ single LB |
| **Use case** | Internal comms | Dev/test | Production (1 service) | Production (multiple services) |

---

## Quick Reference

### Basic Ingress Template

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: basic-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

### TLS Ingress Template

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - example.com
    secretName: example-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

### Multiple Paths Template

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-path-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
```

### Multiple Hosts Template

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
spec:
  rules:
  - host: app1.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  - host: app2.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
```

---

## Summary

**Congratulations!** You've completed Lesson 5 on Kubernetes Ingress.

### What You Learned

✅ **The limitations of Services** for web applications  
✅ **What Ingress is** and why it's essential  
✅ **Path-based routing** - route by URL path  
✅ **Host-based routing** - route by hostname  
✅ **TLS/HTTPS configuration** - secure your applications  
✅ **Ingress annotations** - customize behavior  
✅ **Troubleshooting techniques** - debug Ingress issues  

### Skills Acquired

- Configure complex routing rules
- Expose multiple services through single entry point
- Implement SSL/TLS termination
- Use annotations for advanced features
- Debug and troubleshoot Ingress issues
- Understand when to use Ingress vs Services

### Production Readiness

You can now:
- Design and implement production-grade HTTP/HTTPS routing
- Reduce cloud costs by using single LoadBalancer
- Configure secure applications with TLS
- Implement multi-tenant applications
- Create API gateways with versioning
- Set up canary deployments

---

## What's Next?

**Upcoming Lessons:**
- **Lesson 6**: Persistent Volumes & Storage - stateful applications
- **Lesson 7**: ConfigMaps & Secrets (Advanced) - configuration management
- **Lesson 8**: StatefulSets - databases and stateful apps
- **Lesson 9**: DaemonSets & Jobs - batch processing and node agents
- **Lesson 10**: RBAC & Security - access control and security

**Recommended Practice:**
1. Complete all homework tasks
2. Build a multi-tier web application
3. Experiment with different annotations
4. Try other Ingress Controllers (Traefik, Kong)
5. Set up cert-manager for automatic TLS

---

**Happy Learning! 🚀**

Ready to move to Lesson 6?