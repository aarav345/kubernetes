# Kubernetes Learning Guide - Lessons 1 & 2

## Table of Contents
- [Lesson 1: Introduction to Kubernetes & Setup](#lesson-1-introduction-to-kubernetes--setup)
- [Lesson 2: Pods - The Smallest Deployable Unit](#lesson-2-pods---the-smallest-deployable-unit)
- [ConfigMaps Deep Dive](#configmaps-deep-dive)

---

## Lesson 1: Introduction to Kubernetes & Setup

### Theory

#### What is Kubernetes?

Kubernetes (K8s) is an open-source container orchestration platform originally developed by Google. It automates the deployment, scaling, and management of containerized applications.

#### Why Kubernetes?

Imagine you have 10 microservices running in Docker containers. Manually managing them means:
- Starting/stopping containers manually
- No automatic restart if a container crashes
- Manual load balancing between replicas
- No easy way to update without downtime
- Difficult to scale based on load

Kubernetes solves all of these problems automatically.

#### Key Concepts

1. **Cluster**: A set of machines (nodes) that run containerized applications
2. **Node**: A physical or virtual machine in your cluster
3. **Control Plane**: The brain of Kubernetes that makes decisions about the cluster
4. **Worker Nodes**: Machines that run your actual applications

#### Architecture Overview

```
Control Plane Components:
- API Server: Frontend for Kubernetes control plane
- etcd: Key-value store for cluster data
- Scheduler: Assigns pods to nodes
- Controller Manager: Runs controller processes

Worker Node Components:
- kubelet: Agent that ensures containers are running
- kube-proxy: Network proxy
- Container Runtime: Docker/containerd
```

### Practical Lab

**Objective**: Set up a local Kubernetes cluster

#### Step 1: Install Minikube

**For Mac:**
```bash
brew install minikube
```

**For Linux:**
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

**For Windows:**
Download from https://minikube.sigs.k8s.io/docs/start/

#### Step 2: Install kubectl

**For Mac:**
```bash
brew install kubectl
```

**For Linux:**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

#### Step 3: Start Minikube

```bash
minikube start
```

This creates a single-node Kubernetes cluster on your local machine.

#### Step 4: Verify Installation

```bash
# Check cluster status
kubectl cluster-info

# View nodes
kubectl get nodes

# Check all system pods
kubectl get pods -n kube-system
```

**Expected Output:**
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   1m    v1.28.3
```

### Exercises

1. **Check your kubectl version**: `kubectl version --short`
2. **Get detailed info about your node**: `kubectl describe node minikube`
3. **List all namespaces**: `kubectl get namespaces`

### Key Takeaways

- Kubernetes orchestrates containers across multiple machines
- The control plane manages the cluster state
- Worker nodes run your applications
- kubectl is your main tool for interacting with Kubernetes
- Minikube creates a local single-node cluster for learning

### Homework

Before the next lesson:
1. Ensure Minikube is running: `minikube status`
2. Familiarize yourself with kubectl: `kubectl help`
3. Read about Docker containers if you're not familiar with them

---

## Lesson 2: Pods - The Smallest Deployable Unit

### Theory

#### What is a Pod?

A Pod is the smallest deployable unit in Kubernetes. It represents a single instance of a running process in your cluster.

#### Key Characteristics

1. **Container Wrapper**: A Pod wraps one or more containers
2. **Shared Resources**: Containers in a Pod share:
   - Network namespace (same IP address)
   - Storage volumes
   - IPC namespace
3. **Atomic Unit**: Pods are created and destroyed as a single unit
4. **Ephemeral**: Pods are mortal - when they die, they're gone forever

#### Single vs Multi-Container Pods

```
Single Container Pod (Most Common):
[Pod]
 └── [Container: nginx]

Multi-Container Pod (Sidecar Pattern):
[Pod]
 ├── [Main Container: web-app]
 └── [Sidecar Container: log-collector]
```

#### Pod Lifecycle States

- **Pending**: Pod accepted but containers not yet created
- **Running**: Pod bound to node, at least one container running
- **Succeeded**: All containers terminated successfully
- **Failed**: All containers terminated, at least one failed
- **Unknown**: Pod state cannot be determined

### Practical Lab

#### Lab 1: Create Your First Pod

Create a file called `nginx-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    environment: development
spec:
  containers:
  - name: nginx-container
    image: nginx:latest
    ports:
    - containerPort: 80
```

**Understanding the YAML:**
- `apiVersion`: Which version of the Kubernetes API
- `kind`: Type of object (Pod, Deployment, Service, etc.)
- `metadata`: Data about the object (name, labels)
- `spec`: Desired state specification

**Deploy the Pod:**

```bash
# Create the pod
kubectl apply -f nginx-pod.yaml

# Verify it's running
kubectl get pods

# Get detailed information
kubectl describe pod nginx-pod

# Check logs
kubectl logs nginx-pod
```

#### Lab 2: Interact with Your Pod

```bash
# Execute a command in the pod
kubectl exec nginx-pod -- nginx -v

# Get an interactive shell
kubectl exec -it nginx-pod -- /bin/bash
# Inside the container, try:
# curl localhost
# exit

# Port-forward to access from your machine
kubectl port-forward nginx-pod 8080:80

# Open browser to http://localhost:8080
# You should see "Welcome to nginx!"
```

#### Lab 3: Multi-Container Pod

Create `multi-container-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
  
  - name: log-sidecar
    image: busybox
    command: ["sh", "-c", "tail -f /logs/access.log"]
    volumeMounts:
    - name: shared-logs
      mountPath: /logs
  
  volumes:
  - name: shared-logs
    emptyDir: {}
```

```bash
# Deploy the multi-container pod
kubectl apply -f multi-container-pod.yaml

# Check both containers are running
kubectl get pod multi-container-pod

# View logs from specific container
kubectl logs multi-container-pod -c nginx
kubectl logs multi-container-pod -c log-sidecar
```

#### Lab 4: Pod with Environment Variables

Create `pod-with-env.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
spec:
  containers:
  - name: env-container
    image: busybox
    command: ["sh", "-c", "echo Hello $USER_NAME! Your role is $USER_ROLE && sleep 3600"]
    env:
    - name: USER_NAME
      value: "Kubernetes Learner"
    - name: USER_ROLE
      value: "Developer"
```

```bash
kubectl apply -f pod-with-env.yaml
kubectl logs env-pod
```

### Exercises

1. **Create a Pod running Redis**:
   - Use image `redis:alpine`
   - Name it `redis-pod`
   - Expose port 6379

2. **Check Resource Usage**:
   ```bash
   kubectl top pod nginx-pod
   ```

3. **Delete and Recreate**:
   ```bash
   kubectl delete pod nginx-pod
   kubectl apply -f nginx-pod.yaml
   # Notice it gets a new IP address
   ```

4. **Create a Pod Imperatively** (without YAML):
   ```bash
   kubectl run busybox-pod --image=busybox --command -- sleep 3600
   kubectl get pods
   ```

### Troubleshooting Commands

```bash
# Get pod details
kubectl describe pod <pod-name>

# View logs
kubectl logs <pod-name>
kubectl logs <pod-name> -f  # Follow logs

# Get YAML of running pod
kubectl get pod <pod-name> -o yaml

# Delete a pod
kubectl delete pod <pod-name>

# Delete all pods
kubectl delete pods --all
```

### Key Takeaways

- Pods are the atomic unit in Kubernetes
- Each Pod gets its own IP address
- Containers in a Pod share network and storage
- Pods are ephemeral - they can be killed and recreated
- Use `kubectl apply` for declarative management
- Labels help organize and select resources

### Homework

1. Create a Pod running PostgreSQL with environment variables for username and password
2. Practice exec-ing into different pods
3. Experiment with `kubectl get pods -o wide` to see more details
4. Try deleting a pod and watch it disappear with `kubectl get pods -w` (-w for watch mode)

---

#### Homework solution

Create `postgressql-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgresql-pod
  labels:
    app: postgresql
    tier: database


spec:
  containers:
    - name: postgresql-pod
      image: postgres:16
      ports:
        - containerPort: 5432
      env:
        - name: POSTGRES_USER
          value: postgres
        - name: POSTGRES_PASSWORD
          value: postgres
        - name: POSTGRES_DB
          value: postgres
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
      
      volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data/pgdata
  
  volumes:
    - name: postgres-data
      emptyDir: {}
```



```bash
kubectl apply -f postgressql-pod.yaml
kubectl exec -it postgresql-pod -- psql -U postgres -d postgres
kubectl get pods -w # (watch the pods)
kubectl logs postgresql-pod
```


### Problems in alpine image of postgresql

The postgres:alpine image runs as user ID 70 (postgres user), but the emptyDir volume is owned by root. 
We need to tell Kubernetes to set the correct permissions.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgresql-pod
  labels:
    app: postgresql
    tier: database

spec:
  securityContext:
    fsGroup: 70        # Set group ownership to postgres group
    runAsUser: 70      # Run as postgres user
    runAsNonRoot: true

```