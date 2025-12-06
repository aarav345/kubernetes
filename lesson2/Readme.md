# Kubernetes Lesson 3: ReplicaSets and Deployments

## Table of Contents
- [Overview](#overview)
- [Theory](#theory)
  - [The Problem with Bare Pods](#the-problem-with-bare-pods)
  - [ReplicaSets](#replicasets)
  - [Deployments](#deployments)
  - [Rolling Updates](#rolling-updates)
- [Practical Labs](#practical-labs)
- [Common Commands](#common-commands)
- [Key Takeaways](#key-takeaways)
- [Homework](#homework)

---

## Overview

This lesson covers Kubernetes controllers that manage Pods at scale:
- **ReplicaSets**: Ensure a specified number of pod replicas are running
- **Deployments**: Manage ReplicaSets and provide rolling updates and rollbacks

**Learning Objectives:**
- Understand why bare Pods are insufficient for production
- Learn how ReplicaSets provide self-healing and scaling
- Master Deployments for zero-downtime updates
- Practice scaling, updating, and rolling back applications

---

## Theory

### The Problem with Bare Pods

Creating Pods directly has critical limitations:

```bash
kubectl run nginx --image=nginx
kubectl delete pod nginx
kubectl get pods
# No pods found! ❌ It's gone forever!
```

**Issues with Bare Pods:**
- ❌ No automatic restart if crashed
- ❌ Lost forever if node fails
- ❌ Manual scaling (create multiple YAML files)
- ❌ Manual updates (delete and recreate)
- ❌ No load distribution

**Solution:** Use Controllers (ReplicaSets, Deployments)

### ReplicaSets

A **ReplicaSet** ensures a specified number of pod replicas are running at all times.

**Architecture:**
```
┌─────────────────────────────────────┐
│         ReplicaSet                  │
│  Desired: 3 replicas                │
│  Selector: app=nginx                │
└─────────────────────────────────────┘
              │
    ┌─────────┼─────────┐
    ▼         ▼         ▼
┌───────┐ ┌───────┐ ┌───────┐
│ Pod 1 │ │ Pod 2 │ │ Pod 3 │
│app=   │ │app=   │ │app=   │
│nginx  │ │nginx  │ │nginx  │
└───────┘ └───────┘ └───────┘
```

**Key Features:**
- ✅ **Self-Healing**: Automatically replaces failed pods
- ✅ **Scaling**: Easily scale up/down
- ✅ **Load Distribution**: Multiple pod replicas
- ✅ **Label Selector**: Uses labels to identify pods

**Basic ReplicaSet YAML:**
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
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
        image: nginx:1.25
        ports:
        - containerPort: 80
```

**Important:** Selector labels must match template labels!

### Deployments

A **Deployment** is a higher-level abstraction that manages ReplicaSets.

**Why Deployments > ReplicaSets:**

| Feature | ReplicaSet | Deployment |
|---------|------------|------------|
| Self-healing | ✅ | ✅ |
| Scaling | ✅ | ✅ |
| Rolling updates | ❌ | ✅ |
| Rollback | ❌ | ✅ |
| Version history | ❌ | ✅ |
| **Use in production?** | ❌ | ✅ |

**Deployment Architecture:**
```
┌──────────────────────────────────────────┐
│           Deployment                     │
│  Strategy: RollingUpdate                 │
│  Replicas: 3                             │
└──────────────────────────────────────────┘
              │
    ┌─────────┴─────────┐
    ▼                   ▼
┌─────────────┐   ┌─────────────┐
│ ReplicaSet  │   │ ReplicaSet  │
│ (old v1.0)  │   │ (new v2.0)  │
└─────────────┘   └─────────────┘
     │                  │
  (scaling          (scaling
   down)              up)
```

**Basic Deployment YAML:**
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
        image: nginx:1.25
        ports:
        - containerPort: 80
```

### Rolling Updates

When updating a Deployment, Kubernetes performs a **rolling update**:

```
Initial: 3 pods v1.0
┌─────┐ ┌─────┐ ┌─────┐
│ v1  │ │ v1  │ │ v1  │
└─────┘ └─────┘ └─────┘

Step 1: Create 1 new pod (v2.0)
┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
│ v1  │ │ v1  │ │ v1  │ │ v2  │
└─────┘ └─────┘ └─────┘ └─────┘

Step 2: Delete 1 old pod
┌─────┐ ┌─────┐ ┌─────┐
│ v1  │ │ v1  │ │ v2  │
└─────┘ └─────┘ └─────┘

Step 3: Repeat...
┌─────┐ ┌─────┐ ┌─────┐
│ v1  │ │ v2  │ │ v2  │
└─────┘ └─────┘ └─────┘

Final: All pods v2.0
┌─────┐ ┌─────┐ ┌─────┐
│ v2  │ │ v2  │ │ v2  │
└─────┘ └─────┘ └─────┘
```

**Result:** Zero downtime! Always have available pods.

**Update Strategies:**

1. **RollingUpdate (Default)** - Gradual replacement
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # Max extra pods during update
    maxUnavailable: 1  # Max pods that can be unavailable
```

2. **Recreate** - Kill all old pods, then create new (causes downtime)
```yaml
strategy:
  type: Recreate
```

---

## Practical Labs

### Lab 1: Problem with Bare Pods

```bash
# Create a pod
kubectl run test-pod --image=nginx

# Delete it
kubectl delete pod test-pod

# It's gone forever!
kubectl get pods
```

### Lab 2: Create a ReplicaSet

**nginx-replicaset.yaml:**
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
  labels:
    app: nginx
    tier: frontend
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
        image: nginx:1.25
        ports:
        - containerPort: 80
```

```bash
# Create ReplicaSet
kubectl apply -f nginx-replicaset.yaml

# View ReplicaSets
kubectl get replicasets
kubectl get rs

# View pods
kubectl get pods

# Output shows 3 pods with names like:
# nginx-replicaset-abc12
# nginx-replicaset-def34
# nginx-replicaset-ghi56
```

### Lab 3: ReplicaSet Self-Healing

```bash
# List pods
kubectl get pods

# Delete one pod
kubectl delete pod nginx-replicaset-abc12

# Immediately check - a new pod is already being created!
kubectl get pods

# Watch in real-time
kubectl get pods -w
# In another terminal, delete a pod
```

### Lab 4: Scaling a ReplicaSet

**Method 1: Edit YAML**
```bash
# Edit replicas: 3 to replicas: 5
kubectl apply -f nginx-replicaset.yaml
kubectl get pods
```

**Method 2: Imperative (faster)**
```bash
# Scale to 5
kubectl scale replicaset nginx-replicaset --replicas=5
kubectl get rs

# Scale down to 2
kubectl scale replicaset nginx-replicaset --replicas=2
kubectl get pods -w

# Scale back to 3
kubectl scale replicaset nginx-replicaset --replicas=3
```

### Lab 5: Create a Deployment

**nginx-deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
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
        image: nginx:1.25
        ports:
        - containerPort: 80
```

```bash
# Clean up ReplicaSet first
kubectl delete replicaset nginx-replicaset

# Create Deployment
kubectl apply -f nginx-deployment.yaml

# View Deployment
kubectl get deployments
kubectl get deploy

# View ReplicaSet created by Deployment
kubectl get replicasets

# View Pods
kubectl get pods

# View all together
kubectl get deploy,rs,pods
```

**Pod Naming Convention:**
```
nginx-deployment-7d4b8d5f9c-abc12
     ↓              ↓          ↓
Deployment    ReplicaSet   Pod
   name        hash ID    random
```

### Lab 6: Scaling Deployments

```bash
# Scale up to 5
kubectl scale deployment nginx-deployment --replicas=5

# View all resources
kubectl get deploy,rs,pods

# Scale down to 2
kubectl scale deployment nginx-deployment --replicas=2
kubectl get pods -w

# Scale back to 3
kubectl scale deployment nginx-deployment --replicas=3
```

### Lab 7: Rolling Update

```bash
# Check current image
kubectl describe deployment nginx-deployment | grep Image

# Update image from 1.25 to 1.26
kubectl set image deployment/nginx-deployment nginx=nginx:1.26

# Watch the rollout
kubectl rollout status deployment/nginx-deployment

# Output:
# Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3...
# Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3...
# deployment "nginx-deployment" successfully rolled out

# Watch pods during update (another terminal)
kubectl get pods -w
# You'll see new pods created and old ones terminated
```

**Alternative: Update via YAML**
```bash
# Edit nginx-deployment.yaml
# Change image: nginx:1.25 to image: nginx:1.26

kubectl apply -f nginx-deployment.yaml
kubectl rollout status deployment/nginx-deployment
```

### Lab 8: Rollout History and Rollback

```bash
# View rollout history
kubectl rollout history deployment/nginx-deployment

# Output:
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>

# Get details of specific revision
kubectl rollout history deployment/nginx-deployment --revision=2

# Rollback to previous version
kubectl rollout undo deployment/nginx-deployment

# Watch rollback
kubectl rollout status deployment/nginx-deployment

# Rollback to specific revision
kubectl rollout undo deployment/nginx-deployment --to-revision=1

# Verify image version
kubectl describe deployment nginx-deployment | grep Image
```

### Lab 9: Deployment with Resource Limits

**nginx-deployment-resources.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment-resources
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-resources
  template:
    metadata:
      labels:
        app: nginx-resources
    spec:
      containers:
      - name: nginx
        image: nginx:1.26
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

**Understanding Resources:**
- **requests**: Minimum guaranteed resources
- **limits**: Maximum allowed resources
- **cpu**: "250m" = 0.25 cores, "1000m" = 1 core
- **memory**: "64Mi" = 64 Mebibytes

```bash
kubectl apply -f nginx-deployment-resources.yaml

# Check resources
kubectl describe deployment nginx-deployment-resources
kubectl top pods
```

### Lab 10: Update Strategies

**RollingUpdate Strategy:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-rolling
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: nginx-rolling
  template:
    metadata:
      labels:
        app: nginx-rolling
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

**Recreate Strategy:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-recreate
spec:
  replicas: 3
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nginx-recreate
  template:
    metadata:
      labels:
        app: nginx-recreate
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

```bash
# Create both
kubectl apply -f nginx-rolling.yaml
kubectl apply -f nginx-recreate.yaml

# Update rolling (gradual)
kubectl set image deployment/nginx-rolling nginx=nginx:1.26
kubectl get pods -l app=nginx-rolling -w

# Update recreate (all at once - downtime)
kubectl set image deployment/nginx-recreate nginx=nginx:1.26
kubectl get pods -l app=nginx-recreate -w
```

### Lab 11: Pause and Resume Rollouts

```bash
# Pause deployment
kubectl rollout pause deployment/nginx-deployment

# Make multiple changes (no immediate update)
kubectl set image deployment/nginx-deployment nginx=nginx:1.27
kubectl set resources deployment/nginx-deployment -c nginx --limits=cpu=200m,memory=256Mi

# Nothing happens yet!

# Resume to apply all changes
kubectl rollout resume deployment/nginx-deployment
kubectl rollout status deployment/nginx-deployment
```

---

## Common Commands

### Deployment Management

```bash
# Create
kubectl create deployment <name> --image=<image> --replicas=<n>
kubectl apply -f deployment.yaml

# View
kubectl get deployments
kubectl get deploy
kubectl get deploy -o wide
kubectl describe deployment <name>

# Scale
kubectl scale deployment <name> --replicas=<n>

# Update Image
kubectl set image deployment/<name> <container>=<new-image>

# Rollout
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>
kubectl rollout undo deployment/<name> --to-revision=<n>
kubectl rollout pause deployment/<name>
kubectl rollout resume deployment/<name>
kubectl rollout restart deployment/<name>

# Delete
kubectl delete deployment <name>
kubectl delete -f deployment.yaml

# Edit
kubectl edit deployment <name>

# Export YAML
kubectl get deployment <name> -o yaml > deployment-backup.yaml
```

### ReplicaSet Management

```bash
# View
kubectl get replicasets
kubectl get rs
kubectl describe rs <name>

# Scale
kubectl scale replicaset <name> --replicas=<n>

# Delete
kubectl delete replicaset <name>
```

### Useful Combinations

```bash
# View everything together
kubectl get deploy,rs,pods
kubectl get all

# Watch resources
kubectl get pods -w
kubectl get deploy -w

# Filter by labels
kubectl get pods -l app=nginx
kubectl get pods -l app=nginx,tier=frontend

# Resource usage
kubectl top pods
kubectl top nodes
```

---

## Key Takeaways

### Core Concepts

✅ **Never use bare Pods in production** - they don't self-heal  
✅ **ReplicaSets ensure desired replicas** - but don't create them directly  
✅ **Deployments are the standard** - they manage ReplicaSets  
✅ **Rolling updates = zero downtime** - gradual pod replacement  
✅ **Rollback instantly** - when updates go wrong  
✅ **Scaling is simple** - change replica count  
✅ **Self-healing** - pods auto-recreate if they crash  
✅ **Declarative management** - describe desired state  

### When to Use What?

| Scenario | Use |
|----------|-----|
| Production workload | **Deployment** ✅ |
| Stateless application | **Deployment** ✅ |
| Need rolling updates | **Deployment** ✅ |
| Need rollback | **Deployment** ✅ |
| Testing/learning | Pod (temporary) |
| **Never use directly** | ReplicaSet ❌ |

### Best Practices

1. **Always use Deployments** for stateless applications
2. **Set resource requests and limits** for predictable behavior
3. **Use labels consistently** for organization and selection
4. **Keep deployment history** with meaningful annotations
5. **Test updates** in non-production first
6. **Use rolling updates** for zero downtime
7. **Monitor rollouts** with `kubectl rollout status`
8. **Have rollback plan** before updating production

---

## Homework

### Task 1: Create a Multi-Replica Web Application

Create a Deployment for `hashicorp/http-echo`:
- 4 replicas
- Args: `["-text=Hello from Kubernetes!"]`
- Port: 5678

**Starter Template:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: http-echo-deployment
spec:
  replicas: 4
  selector:
    matchLabels:
      app: http-echo
  template:
    metadata:
      labels:
        app: http-echo
    spec:
      containers:
      - name: http-echo
        image: hashicorp/http-echo
        args:
        - "-text=Hello from Kubernetes!"
        ports:
        - containerPort: 5678
```

```bash
kubectl apply -f http-echo-deployment.yaml
kubectl get deploy,rs,pods
```

### Task 2: Practice Scaling

```bash
# Scale to 6 replicas
kubectl scale deployment http-echo-deployment --replicas=6

# Watch pods scale up
kubectl get pods -w

# Scale down to 3
kubectl scale deployment http-echo-deployment --replicas=3

# Verify
kubectl get deploy
```

### Task 3: Update and Rollback

```bash
# Update the text
kubectl set image deployment/http-echo-deployment http-echo=hashicorp/http-echo:latest

# OR edit the args in YAML:
# args: ["-text=Hello from Deployment v2!"]

# Watch rollout
kubectl rollout status deployment/http-echo-deployment

# Check history
kubectl rollout history deployment/http-echo-deployment

# Rollback
kubectl rollout undo deployment/http-echo-deployment

# Verify old text is back
kubectl describe deployment http-echo-deployment
```

### Task 4: Delete Pod and Watch Recovery

```bash
# List pods
kubectl get pods

# Delete one
kubectl delete pod <pod-name>

# Watch it recreate
kubectl get pods -w
```

### Task 5: Deployment with Resource Limits

Create nginx Deployment with:
- 3 replicas
- CPU request: 100m, limit: 200m
- Memory request: 128Mi, limit: 256Mi

**Template:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-with-limits
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-limited
  template:
    metadata:
      labels:
        app: nginx-limited
    spec:
      containers:
      - name: nginx
        image: nginx:1.26
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
        ports:
        - containerPort: 80
```

### Task 6: Compare Update Strategies

1. Create two deployments (RollingUpdate vs Recreate)
2. Update both to new image version
3. Watch the difference: `kubectl get pods -w`
4. Document which has downtime

**Questions to Answer:**
- Which strategy maintains availability?
- Which strategy is faster?
- When would you use Recreate?

---

## Troubleshooting

### Common Issues

**Pods not starting:**
```bash
kubectl describe deployment <name>
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

**Update stuck:**
```bash
kubectl rollout status deployment/<name>
kubectl describe deployment <name>
# Look for events showing why pods aren't ready
```

**Rollback not working:**
```bash
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name> --to-revision=<n>
```

**Scaling issues:**
```bash
kubectl get events --sort-by='.lastTimestamp'
# Check for resource constraints
kubectl describe nodes
```

---

## Clean Up

```bash
# Delete all deployments from this lesson
kubectl delete deployment nginx-deployment
kubectl delete deployment nginx-deployment-resources
kubectl delete deployment nginx-rolling
kubectl delete deployment nginx-recreate
kubectl delete deployment http-echo-deployment
kubectl delete deployment nginx-with-limits

# Verify cleanup
kubectl get deploy,rs,pods
```

---

## What's Next?

**You now know:**
- ✅ How to manage pods at scale with ReplicaSets
- ✅ How to use Deployments for production workloads
- ✅ How to perform zero-downtime rolling updates
- ✅ How to rollback when things go wrong
- ✅ How to scale applications up and down

**Next Lesson: Services & Networking**
- How to expose Deployments to the network
- Service types (ClusterIP, NodePort, LoadBalancer)
- Service discovery and DNS
- Load balancing across pods

---

## Additional Resources

- [Kubernetes Deployments Official Docs](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [ReplicaSet Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
- [Update Strategies](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

---

**Happy Learning! 🚀**