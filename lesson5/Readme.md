# Kubernetes Lesson 6: Persistent Volumes & Storage

## Table of Contents
- [Overview](#overview)
- [Theory](#theory)
  - [The Storage Problem](#the-storage-problem)
  - [Storage Concepts](#storage-concepts)
  - [Access Modes](#access-modes)
  - [Reclaim Policies](#reclaim-policies)
  - [Static vs Dynamic Provisioning](#static-vs-dynamic-provisioning)
- [Practical Labs](#practical-labs)
- [Common Commands](#common-commands)
- [Key Takeaways](#key-takeaways)
- [Homework](#homework)
- [Troubleshooting](#troubleshooting)

---

## Overview

This lesson covers Kubernetes persistent storage, essential for stateful applications like databases, file storage, and applications that need to retain data across pod restarts.

**Learning Objectives:**
- Understand why persistent storage is needed
- Learn about Volumes, PersistentVolumes (PV), and PersistentVolumeClaims (PVC)
- Master static and dynamic provisioning
- Work with StorageClasses
- Deploy stateful applications with persistent data
- Implement best practices for storage management

---

## Theory

### The Storage Problem

**Scenario: Data Loss with Pod Restarts**

```bash
# Create a pod that writes data
kubectl run test-pod --image=nginx --command -- /bin/sh -c \
  "echo 'Hello' > /data/file.txt && sleep 3600"

# Verify file exists
kubectl exec test-pod -- cat /data/file.txt
# Output: Hello

# Delete the pod
kubectl delete pod test-pod

# Recreate and try to read
kubectl run test-pod --image=nginx --command -- sleep 3600
kubectl exec test-pod -- cat /data/file.txt
# Error: file doesn't exist! ❌
```

**Data is Lost When:**
- Pod is deleted
- Pod crashes and restarts
- Pod is rescheduled to another node
- Container restarts within a pod

### Storage Lifecycle Comparison

**Without Persistent Storage:**
```
Pod Created → Container writes data → Pod Deleted → Data LOST ❌
```

**With Persistent Storage:**
```
Pod Created → Volume attached → Container writes to volume
     ↓
Pod Deleted → Volume persists ✅
     ↓
New Pod Created → Same volume attached → Data still there ✅
```

### Storage Concepts

Kubernetes has a layered storage abstraction:

```
┌─────────────────────────────────────────┐
│  Pod                                    │
│  ┌────────────────────────────────┐    │
│  │ Container                      │    │
│  │  volumeMounts:                 │    │
│  │    - mountPath: /data          │    │
│  │      name: my-storage          │    │
│  └────────────────────────────────┘    │
│                  ↓                      │
│  volumes:                               │
│    - name: my-storage                   │
│      persistentVolumeClaim:             │
│        claimName: my-pvc                │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ PersistentVolumeClaim (PVC)             │
│ - Request for storage                   │
│ - Size: 10Gi                            │
│ - Access mode: ReadWriteOnce            │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ PersistentVolume (PV)                   │
│ - Actual storage resource               │
│ - AWS EBS, GCE PD, NFS, etc.            │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ Physical Storage                        │
│ - Cloud disk, NFS server, local disk    │
└─────────────────────────────────────────┘
```

#### 1. Volume

A **Volume** is a directory accessible to containers in a Pod.

**Common Volume Types:**
- **emptyDir**: Empty directory, exists as long as Pod exists
- **hostPath**: Mount directory from host node (⚠️ not for production)
- **configMap**: Mount ConfigMap as files
- **secret**: Mount Secret as files
- **persistentVolumeClaim**: Use a PersistentVolume (recommended)
- **nfs**: Network File System
- **Cloud volumes**: awsElasticBlockStore, gcePersistentDisk, azureDisk

#### 2. PersistentVolume (PV)

A **PersistentVolume** is a piece of storage in the cluster provisioned by an administrator or dynamically provisioned.

**Key Properties:**
- **Capacity**: Size (e.g., 10Gi)
- **Access Modes**: RWO, ROX, RWX
- **Storage Class**: Type/quality of storage
- **Reclaim Policy**: Retain, Delete, Recycle

**PV Lifecycle:**
```
Available → Bound → Released → (Reclaimed)
```

**PV Status:**
- **Available**: Not yet bound to a PVC
- **Bound**: Bound to a PVC
- **Released**: PVC was deleted, but PV not reclaimed yet
- **Failed**: Automatic reclamation failed

#### 3. PersistentVolumeClaim (PVC)

A **PersistentVolumeClaim** is a request for storage by a user.

**Analogy:**
- **PV** = Available apartment
- **PVC** = Your rental application
- **Binding** = You get the apartment

**PVC Specifies:**
- Storage size needed
- Access mode required
- Storage class preference

#### 4. StorageClass

A **StorageClass** describes different types of storage available.

**Examples:**
- `fast-ssd`: High-performance SSD storage
- `standard-hdd`: Standard HDD storage
- `network-storage`: Shared network storage
- `local-storage`: Local node storage

**Benefits:**
- Dynamic provisioning (PVs created automatically)
- Different performance tiers
- Different backup policies
- Cost optimization

### Access Modes

PVs support different access modes:

| Mode | Short | Description | Use Case |
|------|-------|-------------|----------|
| **ReadWriteOnce** | RWO | Mounted read-write by single node | Databases, single-instance apps |
| **ReadOnlyMany** | ROX | Mounted read-only by many nodes | Static content, shared configs |
| **ReadWriteMany** | RWX | Mounted read-write by many nodes | Shared file storage, logs |
| **ReadWriteOncePod** | RWOP | Mounted read-write by single pod | Kubernetes 1.22+ |

**Important:** Not all storage types support all access modes!

```
Storage Type Support:
- AWS EBS:     RWO only ✅
- GCE PD:      RWO only ✅
- NFS:         RWO, ROX, RWX ✅
- Azure Disk:  RWO only ✅
- HostPath:    RWO only ✅ (not for production)
```

### Reclaim Policies

What happens to a PV when its PVC is deleted?

| Policy | Behavior | Use Case |
|--------|----------|----------|
| **Retain** | PV remains, manual cleanup needed | Production data (safest) |
| **Delete** | PV and storage deleted automatically | Development, temporary data |
| **Recycle** | PV data scrubbed, made available again | Deprecated (don't use) |

**Best Practice:** Use `Retain` for production data to prevent accidental deletion.

### Static vs Dynamic Provisioning

#### Static Provisioning

Admin manually creates PVs, users create PVCs to claim them.

```
Step 1: Admin creates PV (10Gi)
Step 2: User creates PVC (needs 5Gi)
Step 3: Kubernetes binds PVC to PV ✅
```

**Pros:**
- Admin has full control
- Can use any storage backend
- Explicit resource allocation

**Cons:**
- Manual work for admins
- PVs must exist before PVCs
- Less flexible

#### Dynamic Provisioning

PVs are created automatically when PVC is created (using StorageClass).

```
Step 1: User creates PVC with StorageClass (needs 5Gi)
Step 2: Kubernetes creates PV automatically ✅
Step 3: Kubernetes binds PVC to new PV ✅
```

**Pros:**
- No manual PV creation
- Automatic provisioning
- More flexible
- Scales better

**Cons:**
- Requires StorageClass setup
- May cost more (cloud)
- Less admin control

### Storage Architecture Diagram

```
                    User/Developer
                          |
                          | Creates PVC
                          ↓
                    ┌──────────────┐
                    │     PVC      │
                    │ Request: 10Gi│
                    └──────────────┘
                          |
        ┌─────────────────┼─────────────────┐
        |                 |                 |
   Static Provisioning   Dynamic Provisioning
        ↓                 ↓
  ┌──────────┐      ┌──────────┐
  │ Pre-     │      │ Storage  │
  │ existing │      │ Class    │
  │ PV       │      └──────────┘
  └──────────┘            |
        |                 | Provisions
        |                 ↓
        |           ┌──────────┐
        └──────────→│    PV    │
                    │ Size: 10Gi│
                    └──────────┘
                          |
                          | Mounts to
                          ↓
                    ┌──────────┐
                    │   Pod    │
                    └──────────┘
```

---

## Practical Labs

### Lab 1: The Problem - emptyDir

Demonstrates data loss with emptyDir volumes.

**emptydir-pod.yaml:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
spec:
  containers:
  - name: writer
    image: busybox
    command: 
    - /bin/sh
    - -c
    - |
      echo "Writing data at $(date)" > /data/file.txt
      cat /data/file.txt
      sleep 3600
    volumeMounts:
    - name: data-volume
      mountPath: /data
  
  volumes:
  - name: data-volume
    emptyDir: {}  # Lost when pod is deleted
```

```bash
# Create pod
kubectl apply -f emptydir-pod.yaml

# Check file
kubectl exec emptydir-pod -- cat /data/file.txt

# Delete and recreate
kubectl delete pod emptydir-pod
kubectl apply -f emptydir-pod.yaml

# File is gone! Different timestamp
kubectl exec emptydir-pod -- cat /data/file.txt
```

### Lab 2: HostPath Volume (Not for Production!)

**⚠️ Warning:** HostPath is NOT recommended for production!

**hostpath-pod.yaml:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
  - name: test
    image: busybox
    command:
    - /bin/sh
    - -c
    - |
      echo "Data from hostpath at $(date)" > /data/file.txt
      cat /data/file.txt
      sleep 3600
    volumeMounts:
    - name: host-volume
      mountPath: /data
  
  volumes:
  - name: host-volume
    hostPath:
      path: /mnt/data
      type: DirectoryOrCreate
```

```bash
kubectl apply -f hostpath-pod.yaml

# Data persists across pod restarts
kubectl exec hostpath-pod -- cat /data/file.txt
kubectl delete pod hostpath-pod
kubectl apply -f hostpath-pod.yaml
kubectl exec hostpath-pod -- cat /data/file.txt
# Same data! ✅
```

**Problems with HostPath:**
- ❌ Data only on specific node
- ❌ Pod rescheduled = data lost
- ❌ Security risk
- ❌ Not portable

### Lab 3: Create a PersistentVolume (Static)

**persistent-volume.yaml:**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
    type: DirectoryOrCreate
```

```bash
# Create PV
kubectl apply -f persistent-volume.yaml

# View PVs
kubectl get pv

# Output:
# NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS
# my-pv   1Gi        RWO            Retain           Available

# Describe PV
kubectl describe pv my-pv
```

### Lab 4: Create a PersistentVolumeClaim

**persistent-volume-claim.yaml:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi  # Less than PV's 1Gi
  storageClassName: manual
```

```bash
# Create PVC
kubectl apply -f persistent-volume-claim.yaml

# View PVCs
kubectl get pvc

# Output:
# NAME     STATUS   VOLUME   CAPACITY   ACCESS MODES
# my-pvc   Bound    my-pv    1Gi        RWO

# PV is now Bound
kubectl get pv

# Output:
# NAME    CAPACITY   STATUS   CLAIM
# my-pv   1Gi        Bound    default/my-pvc
```

**Binding Process:**
```
1. User creates PVC requesting 500Mi
2. K8s finds available PV with ≥ 500Mi
3. K8s binds PVC to PV
4. Both show Status: Bound
```

### Lab 5: Use PVC in a Pod

**pod-with-pvc.yaml:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: app
    image: busybox
    command:
    - /bin/sh
    - -c
    - |
      echo "Persistent data at $(date)" > /data/persistent.txt
      cat /data/persistent.txt
      sleep 3600
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: my-pvc
```

```bash
# Create pod
kubectl apply -f pod-with-pvc.yaml

# Check data
kubectl exec pvc-pod -- cat /data/persistent.txt

# Delete pod
kubectl delete pod pvc-pod

# Recreate
kubectl apply -f pod-with-pvc.yaml

# Data persists! ✅
kubectl exec pvc-pod -- cat /data/persistent.txt
```

### Lab 6: Dynamic Provisioning with StorageClass

**storage-class.yaml:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: k8s.io/minikube-hostpath
parameters:
  type: pd-ssd
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

**dynamic-pvc.yaml:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-storage
  resources:
    requests:
      storage: 2Gi
```

```bash
# Create StorageClass
kubectl apply -f storage-class.yaml

# View StorageClasses
kubectl get storageclass
kubectl get sc

# Create PVC (PV created automatically!)
kubectl apply -f dynamic-pvc.yaml

# Watch PVC bind
kubectl get pvc dynamic-pvc -w

# View auto-created PV
kubectl get pv
```

### Lab 7: Default StorageClass (Minikube)

```bash
# View default StorageClass
kubectl get storageclass

# Output includes:
# NAME                 PROVISIONER
# standard (default)   k8s.io/minikube-hostpath

# Create PVC without storageClassName
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: auto-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  # No storageClassName - uses default
EOF

# PV created automatically
kubectl get pvc auto-pvc
kubectl get pv
```

### Lab 8: StatefulSet with Persistent Storage

**statefulset-with-storage.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  clusterIP: None
  selector:
    app: nginx
  ports:
  - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: nginx-service
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
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
```

```bash
# Create StatefulSet
kubectl apply -f statefulset-with-storage.yaml

# Watch pods create
kubectl get pods -w

# View PVCs - one per pod!
kubectl get pvc

# Output:
# NAME        STATUS   VOLUME      CAPACITY
# www-web-0   Bound    pvc-xxxxx   1Gi
# www-web-1   Bound    pvc-yyyyy   1Gi
# www-web-2   Bound    pvc-zzzzz   1Gi

# Write unique data to each pod
kubectl exec web-0 -- bash -c \
  "echo 'Data from web-0' > /usr/share/nginx/html/index.html"
kubectl exec web-1 -- bash -c \
  "echo 'Data from web-1' > /usr/share/nginx/html/index.html"
kubectl exec web-2 -- bash -c \
  "echo 'Data from web-2' > /usr/share/nginx/html/index.html"

# Delete a pod
kubectl delete pod web-0

# New web-0 created with same PVC
# Data persists!
kubectl exec web-0 -- cat /usr/share/nginx/html/index.html
# Output: Data from web-0 ✅
```

### Lab 9: Database with Persistent Storage

**postgres-with-storage.yaml:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
---
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
        - name: POSTGRES_DB
          value: myapp
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
```

```bash
# Deploy PostgreSQL
kubectl apply -f postgres-with-storage.yaml

# Wait for ready
kubectl wait --for=condition=ready pod -l app=postgres --timeout=120s

# Create data
kubectl exec -it $(kubectl get pod -l app=postgres -o name) -- \
  psql -U postgres -d myapp

# In psql:
CREATE TABLE users (id SERIAL PRIMARY KEY, name VARCHAR(50));
INSERT INTO users (name) VALUES ('Alice'), ('Bob'), ('Charlie');
SELECT * FROM users;
\q

# Delete pod
kubectl delete pod -l app=postgres

# New pod created automatically
kubectl get pods -l app=postgres

# Data persists!
kubectl exec -it $(kubectl get pod -l app=postgres -o name) -- \
  psql -U postgres -d myapp -c "SELECT * FROM users;"
# Shows Alice, Bob, Charlie ✅
```

### Lab 10: Resize PVC (Volume Expansion)

```bash
# Check if expansion allowed
kubectl get storageclass standard -o yaml | grep allowVolumeExpansion

# Edit PVC
kubectl edit pvc my-pvc

# Change storage: 1Gi to storage: 2Gi
# Save and exit

# Check PVC capacity
kubectl get pvc my-pvc
# CAPACITY shows 2Gi

# May need pod restart
kubectl delete pod pvc-pod
kubectl apply -f pod-with-pvc.yaml
```

### Lab 11: Clone a PVC

**clone-pvc.yaml:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cloned-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  dataSource:
    kind: PersistentVolumeClaim
    name: my-pvc
```

```bash
kubectl apply -f clone-pvc.yaml
kubectl get pvc cloned-pvc
# New PVC with copy of data
```

---

## Common Commands

### PersistentVolume Management

```bash
# List PVs
kubectl get pv
kubectl get persistentvolumes

# Describe PV
kubectl describe pv <pv-name>

# Get PV details
kubectl get pv <pv-name> -o yaml

# Delete PV
kubectl delete pv <pv-name>

# Check PV reclaim policy
kubectl get pv <pv-name> -o jsonpath='{.spec.persistentVolumeReclaimPolicy}'

# Change reclaim policy
kubectl patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'

# List all PVs with details
kubectl get pv -o wide

# Watch PV status
kubectl get pv -w
```

### PersistentVolumeClaim Management

```bash
# List PVCs
kubectl get pvc
kubectl get persistentvolumeclaims

# Describe PVC
kubectl describe pvc <pvc-name>

# Get PVC details
kubectl get pvc <pvc-name> -o yaml

# Delete PVC
kubectl delete pvc <pvc-name>

# List PVCs in all namespaces
kubectl get pvc --all-namespaces
kubectl get pvc -A

# Check which PV a PVC is bound to
kubectl get pvc <pvc-name> -o jsonpath='{.spec.volumeName}'

# Check PVC capacity
kubectl get pvc <pvc-name> -o jsonpath='{.status.capacity.storage}'
```

### StorageClass Management

```bash
# List StorageClasses
kubectl get storageclass
kubectl get sc

# Describe StorageClass
kubectl describe sc <sc-name>

# Get StorageClass details
kubectl get sc <sc-name> -o yaml

# Create StorageClass
kubectl apply -f storageclass.yaml

# Delete StorageClass
kubectl delete sc <sc-name>

# Set default StorageClass
kubectl patch storageclass <sc-name> \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Remove default annotation
kubectl patch storageclass <sc-name> \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

### Find Which Pods Use a PVC

```bash
# Method 1: Using kubectl
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName=="<pvc-name>") | .metadata.name'

# Method 2: Describe PVC (shows mounted by)
kubectl describe pvc <pvc-name>

# Method 3: Check all pods in namespace
kubectl get pods -o custom-columns=NAME:.metadata.name,PVC:.spec.volumes[*].persistentVolumeClaim.claimName
```

### Volume Operations

```bash
# Check volume mounts in a pod
kubectl exec <pod-name> -- df -h

# List files in mounted volume
kubectl exec <pod-name> -- ls -la /data

# Check volume usage
kubectl exec <pod-name> -- du -sh /data

# Copy data to/from volume
kubectl cp <pod-name>:/data/file.txt ./local-file.txt
kubectl cp ./local-file.txt <pod-name>:/data/file.txt
```

### Debugging Storage Issues

```bash
# Check PVC events
kubectl get events --field-selector involvedObject.name=<pvc-name>

# Check pod events (for mount issues)
kubectl describe pod <pod-name> | grep -A 10 Events

# Check if PVC is bound
kubectl get pvc <pvc-name> -o jsonpath='{.status.phase}'

# List all storage resources
kubectl get pv,pvc,sc

# Check storage capacity
kubectl get pv -o custom-columns=NAME:.metadata.name,CAPACITY:.spec.capacity.storage,STATUS:.status.phase
```

---

## Key Takeaways

### Core Concepts

✅ **Volumes provide storage** to containers in pods  
✅ **emptyDir is ephemeral** - data lost when pod deleted  
✅ **PVs are cluster resources** - actual storage  
✅ **PVCs are user requests** - request for storage  
✅ **StorageClasses enable** dynamic provisioning  
✅ **Access modes control** how volumes are mounted  
✅ **Reclaim policies determine** data lifecycle  
✅ **StatefulSets provide** stable, persistent storage per pod  

### Volume Types Comparison

| Type | Persistence | Scope | Use Case |
|------|-------------|-------|----------|
| **emptyDir** | Pod lifetime | Single pod | Temp data, cache |
| **hostPath** | Node lifetime | Single node | Testing only |
| **PVC** | Cluster-managed | Cluster-wide | Production data |
| **configMap** | Until deleted | Cluster-wide | Configuration files |
| **secret** | Until deleted | Cluster-wide | Sensitive data |

### Access Modes Quick Reference

| Mode | Abbr | Multi-Pod | Multi-Node | Common Storage Types |
|------|------|-----------|------------|---------------------|
| ReadWriteOnce | RWO | ❌ | ❌ | AWS EBS, GCE PD, Azure Disk |
| ReadOnlyMany | ROX | ✅ | ✅ | NFS, CephFS |
| ReadWriteMany | RWX | ✅ | ✅ | NFS, CephFS, GlusterFS |
| ReadWriteOncePod | RWOP | ❌ | ❌ | CSI volumes (K8s 1.22+) |

### Static vs Dynamic Provisioning

| Aspect | Static | Dynamic |
|--------|--------|---------|
| **PV Creation** | Manual by admin | Automatic by K8s |
| **Setup** | Admin creates PVs first | Configure StorageClass |
| **Flexibility** | Less flexible | More flexible |
| **Use Case** | Small clusters, specific needs | Large clusters, standard needs |
| **Cost** | Predictable | Pay-as-you-go |

### Best Practices

1. **Use PVCs in production** - not emptyDir or hostPath
2. **Choose appropriate access mode** - RWO for databases, RWX for shared files
3. **Set realistic storage requests** - don't over-provision
4. **Use StorageClasses** for dynamic provisioning
5. **Set Retain policy** for important data
6. **Implement backups** - automate backup strategies
7. **Monitor storage usage** - set up alerts for capacity
8. **Use StatefulSets** for stateful applications
9. **Label resources** - organize PVs and PVCs with labels
10. **Test disaster recovery** - practice restoring from backups
11. **Secure access** - use RBAC to control PVC creation
12. **Document storage tiers** - make it clear which StorageClass to use

### When to Use What

**Use emptyDir when:**
- Temporary scratch space
- Cache that can be rebuilt
- Data doesn't need to persist

**Use hostPath when:**
- Testing locally (Minikube)
- Never in production!

**Use PVC when:**
- Production databases
- User uploads
- Application state
- Any data that must persist

**Use StatefulSet when:**
- Database clusters
- Applications needing stable network identity
- Applications needing stable storage
- Ordered deployment/scaling required

---

## Homework

### Task 1: WordPress with MySQL

Create a complete WordPress stack with persistent storage.

**Requirements:**
- MySQL deployment with 5Gi PVC for database
- WordPress deployment with 2Gi PVC for uploads
- Both use persistent storage
- Services to expose both
- Test by creating a blog post, delete pods, verify data persists

**Starter Template:**
```yaml
# MySQL PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
# MySQL Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: rootpassword
        - name: MYSQL_DATABASE
          value: wordpress
        - name: MYSQL_USER
          value: wpuser
        - name: MYSQL_PASSWORD
          value: wppassword
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
# Add WordPress PVC, Deployment, and Services...
```

### Task 2: Multi-Tier Application with Storage

Create an application stack with different storage needs:

**Components:**
- Frontend (nginx) - no persistent storage needed
- Backend API (Node.js/Python) - 1Gi PVC for logs
- Database (PostgreSQL) - 10Gi PVC for data
- Cache (Redis) - 500Mi PVC for persistence

**Requirements:**
1. Each component in separate deployment
2. Appropriate PVCs for each
3. Services to connect components
4. Test data persistence for all stateful components

### Task 3: StorageClass Comparison

Create and compare different storage tiers:

**Create 3 StorageClasses:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: k8s.io/minikube-hostpath
parameters:
  type: ssd
reclaimPolicy: Retain
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-hdd
provisioner: k8s.io/minikube-hostpath
parameters:
  type: standard
reclaimPolicy: Delete
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow-archive
provisioner: k8s.io/minikube-hostpath
parameters:
  type: archive
reclaimPolicy: Retain
```

**Tasks:**
1. Create PVC for each StorageClass
2. Deploy pods using each PVC
3. Document:
   - Provisioning time
   - Access patterns
   - When to use each tier

### Task 4: StatefulSet Database Cluster

Create a 3-replica StatefulSet for a database:

**Requirements:**
- StatefulSet with 3 replicas
- Each replica has its own 5Gi PVC
- Headless service for stable network identity
- Write unique data to each replica
- Delete one pod, verify data persists

**Template:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: mongo
  replicas: 3
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
      - name: mongo
        image: mongo:latest
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: mongo-data
          mountPath: /data/db
  volumeClaimTemplates:
  - metadata:
      name: mongo-data
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi
```

### Task 5: Volume Snapshots (Advanced)

Research and implement volume snapshots:

**Requirements:**
1. Create a PVC with data
2. Take a snapshot of the PVC
3. Restore from snapshot to new PVC
4. Verify data is restored correctly

**Note:** Requires VolumeSnapshot CRDs and CSI driver support.

### Task 6: Backup and Restore Strategy

Implement a complete backup and restore workflow:

**Steps:**
1. Deploy database with data
2. Create backup script (exec into pod, dump database)
3. Store backup in separate PVC
4. Delete everything
5. Restore from backup to new environment
6. Verify data integrity

**Backup Script Example:**
```bash
#!/bin/bash
# Backup PostgreSQL
kubectl exec <postgres-pod> -- pg_dump -U postgres myapp > backup.sql

# Store in backup PVC
kubectl cp backup.sql <backup-pod>:/backups/backup-$(date +%Y%m%d).sql

# Restore
kubectl cp <backup-pod>:/backups/backup-20240101.sql ./restore.sql
kubectl exec -i <new-postgres-pod> -- psql -U postgres myapp < restore.sql
```

### Task 7: Shared Storage with NFS (Advanced)

Set up NFS for ReadWriteMany access:

**Requirements:**
1. Set up NFS server (can use a pod)
2. Create PV pointing to NFS export
3. Create PVC with RWX access
4. Deploy multiple pods writing to same volume
5. Verify all pods can read/write simultaneously

---

## Troubleshooting

### Common Issues and Solutions

#### Issue 1: PVC Stuck in Pending

**Symptoms:**
```bash
kubectl get pvc
# NAME     STATUS    VOLUME   CAPACITY
# my-pvc   Pending   
```

**Debug Steps:**
```bash
# 1. Describe PVC for events
kubectl describe pvc my-pvc

# Look for messages like:
# - "no persistent volumes available"
# - "no volume with required capacity"
# - "no volume with required access mode"

# 2. Check available PVs
kubectl get pv

# 3. Check if PV capacity matches request
kubectl get pv -o custom-columns=NAME:.metadata.name,CAPACITY:.spec.capacity.storage,STATUS:.status.phase

# 4. Check access modes match
kubectl get pv <pv-name> -o jsonpath='{.spec.accessModes}'
kubectl get pvc <pvc-name> -o jsonpath='{.spec.accessModes}'

# 5. Check StorageClass exists
kubectl get sc

# 6. If using dynamic provisioning, check provisioner logs
kubectl logs -n kube-system -l app=storage-provisioner
```

**Common Causes:**
- No available PV with sufficient capacity
- StorageClass doesn't exist or misconfigured
- Access mode mismatch
- No storage provisioner available
- Insufficient node resources

**Solutions:**
```bash
# Create PV manually (static provisioning)
kubectl apply -f persistent-volume.yaml

# Or fix StorageClass
kubectl describe sc <sc-name>

# Check node capacity
kubectl describe nodes | grep -A 5 "Allocated resources"
```

#### Issue 2: Pod Can't Mount Volume

**Symptoms:**
```bash
kubectl get pods
# NAME      READY   STATUS              RESTARTS
# my-pod    0/1     ContainerCreating   0

kubectl describe pod my-pod
# Events:
# Warning  FailedMount  Unable to attach or mount volumes
```

**Debug Steps:**
```bash
# 1. Check pod events
kubectl describe pod <pod-name> | grep -A 20 Events

# 2. Check if PVC exists
kubectl get pvc <pvc-name>

# 3. Check if PVC is bound
kubectl get pvc <pvc-name> -o jsonpath='{.status.phase}'

# 4. Check PVC namespace matches pod namespace
kubectl get pvc -n <namespace>

# 5. For RWO volumes, check if already mounted
kubectl get pods -o json | \
  jq -r '.items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName=="<pvc-name>") | .metadata.name'

# 6. Check node has access to storage
kubectl describe node <node-name>
```

**Common Causes:**
- PVC doesn't exist
- PVC in different namespace
- Volume already mounted by another pod (RWO)
- Node doesn't have access to storage backend
- Storage class provisioner issue

**Solutions:**
```bash
# Ensure PVC is created and bound
kubectl get pvc <pvc-name>

# If RWO and already mounted, delete other pod first
kubectl delete pod <other-pod>

# Check PVC is in same namespace as pod
kubectl get pvc -n <pod-namespace>
```

#### Issue 3: Data Not Persisting

**Symptoms:**
- Data disappears after pod restart
- Files not found in expected location

**Debug Steps:**
```bash
# 1. Verify volume is mounted
kubectl exec <pod-name> -- df -h

# 2. Check mount path in container
kubectl exec <pod-name> -- ls -la /data

# 3. Verify PVC is used (not emptyDir)
kubectl get pod <pod-name> -o yaml | grep -A 5 volumes

# 4. Check if writing to correct path
kubectl exec <pod-name> -- mount | grep /data

# 5. Verify PVC exists and is bound
kubectl get pvc
```

**Common Causes:**
- Using emptyDir instead of PVC
- Writing to wrong path (not mounted volume)
- Volume not mounted correctly
- Using hostPath on multi-node cluster (pod moved nodes)

**Solutions:**
```yaml
# Ensure using PVC, not emptyDir
volumes:
- name: storage
  persistentVolumeClaim:
    claimName: my-pvc  # ✅ Correct
# NOT:
# emptyDir: {}  # ❌ Wrong

# Verify mount path matches volumeMount
volumeMounts:
- name: storage
  mountPath: /data  # Must match where app writes
```

#### Issue 4: PV Not Released

**Symptoms:**
```bash
kubectl get pv
# NAME    STATUS     CLAIM
# my-pv   Released   default/old-pvc
```

**Debug Steps:**
```bash
# 1. Check PV status
kubectl describe pv my-pv

# 2. Check reclaim policy
kubectl get pv my-pv -o jsonpath='{.spec.persistentVolumeReclaimPolicy}'

# 3. Check if claimRef still exists
kubectl get pv my-pv -o yaml | grep claimRef -A 5
```

**Solutions:**
```bash
# Option 1: Remove claimRef to make available again
kubectl patch pv my-pv -p '{"spec":{"claimRef": null}}'

# Option 2: Delete and recreate PV
kubectl delete pv my-pv
kubectl apply -f persistent-volume.yaml

# Option 3: Change to Retain policy (if Delete)
kubectl patch pv my-pv -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

#### Issue 5: Volume Expansion Not Working

**Symptoms:**
- PVC shows old capacity after resize
- Application still sees old size

**Debug Steps:**
```bash
# 1. Check if StorageClass allows expansion
kubectl get sc <sc-name> -o jsonpath='{.allowVolumeExpansion}'

# 2. Check PVC status
kubectl describe pvc <pvc-name>

# 3. Check for expansion conditions
kubectl get pvc <pvc-name> -o yaml | grep -A 5 conditions

# 4. Check if pod needs restart
kubectl get pods
```

**Solutions:**
```bash
# 1. Enable expansion on StorageClass
kubectl patch sc <sc-name> -p '{"allowVolumeExpansion": true}'

# 2. Restart pod to see new size (for some filesystems)
kubectl delete pod <pod-name>

# 3. Check filesystem supports online expansion
# ext4: yes
# xfs: yes
# NTFS: needs restart
```

#### Issue 6: Permission Denied on Volume

**Symptoms:**
```bash
kubectl logs <pod-name>
# Error: Permission denied writing to /data
```

**Debug Steps:**
```bash
# 1. Check volume permissions
kubectl exec <pod-name> -- ls -la /data

# 2. Check container user
kubectl exec <pod-name> -- id

# 3. Check pod securityContext
kubectl get pod <pod-name> -o yaml | grep -A 10 securityContext
```

**Solutions:**
```yaml
# Add securityContext to pod
spec:
  securityContext:
    fsGroup: 1000      # Match application user GID
    runAsUser: 1000    # Run as non-root user
  containers:
  - name: app
    image: myapp
    securityContext:
      allowPrivilegeEscalation: false
```

#### Issue 7: Out of Disk Space

**Symptoms:**
```bash
kubectl logs <pod-name>
# No space left on device
```

**Debug Steps:**
```bash
# 1. Check volume usage in pod
kubectl exec <pod-name> -- df -h

# 2. Check PVC capacity
kubectl get pvc <pvc-name>

# 3. Check actual disk usage
kubectl exec <pod-name> -- du -sh /data/*

# 4. Find large files
kubectl exec <pod-name> -- find /data -type f -size +100M
```

**Solutions:**
```bash
# Option 1: Clean up old files
kubectl exec <pod-name> -- rm /data/old-logs/*

# Option 2: Expand volume (if supported)
kubectl edit pvc <pvc-name>
# Increase storage: 10Gi to storage: 20Gi

# Option 3: Add log rotation
# Configure app to rotate/compress logs
```

### Debugging Checklist

```bash
# Quick debugging checklist for storage issues:

□ PVC created? → kubectl get pvc
□ PVC bound? → kubectl get pvc (STATUS should be Bound)
□ PV exists? → kubectl get pv
□ PV bound to correct PVC? → kubectl describe pv
□ Pod created? → kubectl get pods
□ Volume mounted? → kubectl exec <pod> -- df -h
□ Correct mount path? → kubectl exec <pod> -- ls /data
□ Write permissions? → kubectl exec <pod> -- touch /data/test
□ StorageClass exists? → kubectl get sc
□ Sufficient capacity? → kubectl get pvc (check CAPACITY)
□ Access mode correct? → kubectl describe pv | grep "Access Modes"
□ Check events? → kubectl get events --sort-by='.lastTimestamp'
```

---

## Clean Up

```bash
# Delete StatefulSet (keeps PVCs by default)
kubectl delete statefulset web

# Delete StatefulSet and PVCs
kubectl delete statefulset web
kubectl delete pvc www-web-0 www-web-1 www-web-2

# Delete Deployments
kubectl delete deployment postgres

# Delete Services
kubectl delete service nginx-service postgres-service

# Delete PVCs (may delete PVs if reclaimPolicy is Delete)
kubectl delete pvc my-pvc dynamic-pvc auto-pvc postgres-pvc

# Delete PVs manually
kubectl delete pv my-pv

# Delete StorageClass
kubectl delete storageclass fast-storage

# Delete all pods (they'll be recreated by deployments)
kubectl delete pods --all

# Complete cleanup
kubectl delete pvc --all
kubectl delete pv --all
kubectl delete deployment --all
kubectl delete statefulset --all
kubectl delete service --all

# Verify cleanup
kubectl get pv,pvc,pods,deploy,sts,svc
```

---

## Additional Resources

### Official Documentation
- [Kubernetes Storage Documentation](https://kubernetes.io/docs/concepts/storage/)
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)

### Storage Providers
- [AWS EBS CSI Driver](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)
- [GCE PD CSI Driver](https://github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver)
- [Azure Disk CSI Driver](https://github.com/kubernetes-sigs/azuredisk-csi-driver)
- [NFS Provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)
- [Rook (Ceph)](https://rook.io/)

### Tools
- [k9s](https://k9scli.io/) - Terminal UI for Kubernetes
- [Velero](https://velero.io/) - Backup and restore
- [Stash](https://stash.run/) - Backup operator
- [Longhorn](https://longhorn.io/) - Cloud native distributed storage

### Best Practices Guides
- [Storage Best Practices](https://kubernetes.io/docs/concepts/storage/storage-classes/#best-practices)
- [StatefulSet Best Practices](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/)

---

## Quick Reference

### Common YAML Templates

#### Basic PV (Static)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
```

#### Basic PVC
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 5Gi
```

#### Pod with PVC
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-storage
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
```

#### StorageClass
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: k8s.io/minikube-hostpath
parameters:
  type: pd-ssd
reclaimPolicy: Retain
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

#### StatefulSet with volumeClaimTemplates
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
```

---

## Summary

**Congratulations!** You've completed Lesson 6 on Kubernetes Persistent Volumes & Storage.

### What You Learned

✅ **Why persistent storage is essential** for stateful applications  
✅ **Volume types** and when to use each  
✅ **PersistentVolumes (PV)** - cluster storage resources  
✅ **PersistentVolumeClaims (PVC)** - user storage requests  
✅ **StorageClasses** - dynamic provisioning and storage tiers  
✅ **Access modes** - RWO, ROX, RWX  
✅ **Reclaim policies** - Retain, Delete  
✅ **StatefulSets** - stable storage for stateful apps  
✅ **Best practices** for production storage  

### Skills Acquired

- Create and manage PVs and PVCs
- Configure dynamic provisioning with StorageClasses
- Deploy databases with persistent storage
- Use StatefulSets for stateful applications
- Troubleshoot storage issues
- Implement backup and restore strategies

### Production Readiness

You can now:
- Design storage architecture for applications
- Choose appropriate storage types and access modes
- Implement persistent storage for databases
- Configure storage classes for different tiers
- Troubleshoot common storage issues
- Plan backup and disaster recovery strategies

---

## What's Next?

**Upcoming Lessons:**
- **Lesson 7**: ConfigMaps & Secrets (Advanced) - secure configuration
- **Lesson 8**: StatefulSets (Deep Dive) - stateful applications
- **Lesson 9**: DaemonSets, Jobs & CronJobs - batch processing
- **Lesson 10**: RBAC & Security - access control and security

**Recommended Practice:**
1. Complete all homework tasks
2. Deploy a real database with backups
3. Experiment with different StorageClasses
4. Practice disaster recovery scenarios
5. Set up monitoring for storage usage

---

**Happy Learning! 🚀**

Ready to move to Lesson 7 (ConfigMaps & Secrets Advanced)?