# Kubernetes Lesson 7: ConfigMaps & Secrets (Advanced)

## Table of Contents
- [Overview](#overview)
- [Theory](#theory)
  - [ConfigMaps vs Secrets](#configmaps-vs-secrets)
  - [Secret Types](#secret-types)
  - [Consumption Methods](#consumption-methods)
  - [Security Considerations](#security-considerations)
- [Practical Labs](#practical-labs)
- [Security Best Practices](#security-best-practices)
- [Common Commands](#common-commands)
- [Key Takeaways](#key-takeaways)
- [Homework](#homework)
- [Troubleshooting](#troubleshooting)

---

## Overview

This lesson covers advanced usage of ConfigMaps and Secrets for managing application configuration and sensitive data in Kubernetes.

**Learning Objectives:**
- Master ConfigMaps for application configuration
- Understand Secrets and their security implications
- Learn multiple ways to consume configs and secrets
- Implement security best practices
- Work with external secret management
- Handle config updates and rotation

---

## Theory

### Why Separate Configuration from Code?

**The Problem:**
```
❌ Bad Practice:
Application Code → Hard-coded config → Need config change? → Rebuild image → Redeploy

✅ Good Practice:
Application Code (unchanged) → External configuration → Config change? → Just restart pod
```

**Benefits:**
- Same image across environments (dev/staging/prod)
- Quick configuration changes
- No image rebuilds for config updates
- Better security (secrets not in images)
- Easier testing and deployment

### ConfigMaps vs Secrets

| Feature | ConfigMap | Secret |
|---------|-----------|--------|
| **Purpose** | Non-sensitive configuration | Sensitive data |
| **Storage** | Plain text | Base64 encoded |
| **Visibility** | `kubectl get` shows data | `kubectl get` hides data |
| **Encryption** | No (by default) | No (by default)* |
| **Use for** | URLs, flags, settings | Passwords, keys, tokens |
| **Size limit** | 1MB | 1MB |
| **Best practice** | Public config | Private credentials |

*Secrets can be encrypted at rest (optional cluster feature)

**Important:** Base64 is NOT encryption! It's just encoding.

```bash
# Easy to decode
echo "cGFzc3dvcmQ=" | base64 -d
# Output: password
```

### Secret Types

Kubernetes provides several built-in Secret types:

| Type | Purpose | Usage Example |
|------|---------|---------------|
| **Opaque** | Generic secret (default) | Passwords, API keys |
| **kubernetes.io/service-account-token** | Service account token | Auto-created for pods |
| **kubernetes.io/dockerconfigjson** | Docker registry auth | Pull private images |
| **kubernetes.io/tls** | TLS certificate & key | HTTPS Ingress |
| **kubernetes.io/ssh-auth** | SSH authentication | Git operations |
| **kubernetes.io/basic-auth** | Basic authentication | Username/password |

### Consumption Methods

ConfigMaps and Secrets can be consumed in three ways:

```
Method 1: Environment Variables
  ConfigMap/Secret → ENV vars → Container
  ✅ Simple to use
  ❌ Don't update automatically (need pod restart)

Method 2: Command Line Arguments
  ConfigMap/Secret → Args → Container startup command
  ✅ Fine-grained control
  ❌ Don't update automatically

Method 3: Volume Mounts (Files)
  ConfigMap/Secret → Volume → Files in container
  ✅ Auto-update (~60 seconds delay)
  ✅ Can mount specific keys
  ❌ Slightly more complex
```

### ConfigMap/Secret Lifecycle

```
┌─────────────────────────────────────────┐
│ 1. Create ConfigMap/Secret              │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│ 2. Reference in Pod                     │
│    - envFrom / env                       │
│    - volumes                             │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│ 3. Pod uses configuration               │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│ 4. Update ConfigMap/Secret              │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│ 5. Changes propagate                    │
│    - Env vars: Need pod restart ❌       │
│    - Volumes: Auto-update (~60s) ✅      │
└─────────────────────────────────────────┘
```

### Security Considerations

**Secret Security Layers:**

```
Layer 1: RBAC (Access Control)
  └── Control who can read/write Secrets

Layer 2: Encryption at Rest
  └── Secrets encrypted in etcd

Layer 3: Network Security
  └── TLS for API communication

Layer 4: Audit Logging
  └── Track Secret access

Layer 5: External Secret Management
  └── HashiCorp Vault, AWS Secrets Manager
```

**Security Pyramid:**
```
                    ┌─────────────┐
                    │  External   │  (Most secure)
                    │   Secrets   │
                    │  Manager    │
                    └─────────────┘
                  ┌─────────────────┐
                  │   Encryption    │
                  │    at Rest      │
                  └─────────────────┘
              ┌───────────────────────┐
              │       RBAC            │
              │   (Who can access?)   │
              └───────────────────────┘
          ┌───────────────────────────────┐
          │      Kubernetes Secrets       │
          │    (Base64, not encrypted)    │
          └───────────────────────────────┘
```

### Immutable ConfigMaps and Secrets

Since Kubernetes 1.21:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: immutable-config
data:
  version: "1.0.0"
immutable: true  # Cannot be changed!
```

**Benefits:**
- Protects against accidental updates
- Better performance (no watching)
- Forces versioning (create new, not update)
- Safer deployments

---

## Practical Labs

### Lab 1: Create Secrets (Four Methods)

#### Method 1: From Literal Values
```bash
# Create generic secret
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=supersecret123 \
  --from-literal=database=myapp

# View secrets (data hidden)
kubectl get secrets

# Describe (shows keys, not values)
kubectl describe secret db-credentials

# Get as YAML (base64 encoded)
kubectl get secret db-credentials -o yaml
```

#### Method 2: From Files
```bash
# Create credential files
echo -n "admin" > username.txt
echo -n "supersecret123" > password.txt

# Create secret from files
kubectl create secret generic db-creds-from-files \
  --from-file=username=username.txt \
  --from-file=password=password.txt

# Cleanup
rm username.txt password.txt
```

#### Method 3: From YAML with data (Base64)
```yaml
# db-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  # Values must be base64 encoded
  username: YWRtaW4=              # "admin"
  password: c3VwZXJzZWNyZXQxMjM=  # "supersecret123"
```

```bash
# Base64 encode values
echo -n "admin" | base64
# Output: YWRtaW4=

echo -n "supersecret123" | base64
# Output: c3VwZXJzZWNyZXQxMjM=

# Apply
kubectl apply -f db-secret.yaml
```

#### Method 4: From YAML with stringData (Recommended)
```yaml
# db-secret-easy.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret-easy
type: Opaque
stringData:  # No base64 encoding needed!
  username: admin
  password: supersecret123
  database: myapp
```

```bash
kubectl apply -f db-secret-easy.yaml

# Kubernetes auto-converts to base64
kubectl get secret db-secret-easy -o yaml
```

### Lab 2: Use Secrets as Environment Variables

**pod-with-secret-env.yaml:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: app
    image: busybox
    command:
    - sh
    - -c
    - |
      echo "Username: $DB_USER"
      echo "Password: $DB_PASS"
      echo "Database: $DB_NAME"
      sleep 3600
    env:
    # Method 1: Individual keys
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-secret-easy
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-secret-easy
          key: password
    
    # Method 2: All keys at once
    envFrom:
    - secretRef:
        name: db-secret-easy
```

```bash
kubectl apply -f pod-with-secret-env.yaml

# Check logs
kubectl logs secret-env-pod

# Output:
# Username: admin
# Password: supersecret123
# Database: myapp
```

### Lab 3: Mount Secrets as Files

**pod-with-secret-volume.yaml:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
  - name: app
    image: busybox
    command:
    - sh
    - -c
    - |
      echo "=== Secret Files ==="
      ls -la /etc/secrets/
      echo "Username: $(cat /etc/secrets/username)"
      echo "Password: $(cat /etc/secrets/password)"
      sleep 3600
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true  # Best practice!
  
  volumes:
  - name: secret-volume
    secret:
      secretName: db-secret-easy
```

```bash
kubectl apply -f pod-with-secret-volume.yaml

# Check logs
kubectl logs secret-volume-pod

# Verify files
kubectl exec secret-volume-pod -- ls -la /etc/secrets/

# Output:
# -rw-r--r-- username
# -rw-r--r-- password
# -rw-r--r-- database
```

### Lab 4: Selective Secret Keys

Mount only specific keys:

**pod-with-selective-secret.yaml:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: selective-secret-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "ls -la /etc/config/ && sleep 3600"]
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/config
  
  volumes:
  - name: secret-volume
    secret:
      secretName: db-secret-easy
      items:
      - key: username
        path: db-username.txt
      - key: password
        path: db-password.txt
      # 'database' key NOT mounted
```

```bash
kubectl apply -f pod-with-selective-secret.yaml

# Check files
kubectl exec selective-secret-pod -- ls /etc/config/

# Output:
# db-username.txt
# db-password.txt
# (database is NOT here)
```

### Lab 5: ConfigMap with Multiple Data Types

**app-configmap.yaml:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # Simple key-value
  app.name: "My Application"
  app.mode: "production"
  log.level: "info"
  
  # Multi-line config file
  app.properties: |
    server.port=8080
    server.host=0.0.0.0
    database.pool.size=10
    cache.enabled=true
  
  # JSON configuration
  features.json: |
    {
      "feature1": true,
      "feature2": false,
      "maxUsers": 1000
    }
  
  # Shell script
  startup.sh: |
    #!/bin/bash
    echo "Starting application..."
    java -jar /app/application.jar
```

```bash
kubectl apply -f app-configmap.yaml

# View ConfigMap
kubectl describe configmap app-config
```

### Lab 6: ConfigMap as Files

**pod-with-config-files.yaml:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-files-pod
spec:
  containers:
  - name: app
    image: busybox
    command:
    - sh
    - -c
    - |
      echo "=== Configuration Files ==="
      ls -la /etc/config/
      echo ""
      cat /etc/config/app.properties
      sleep 3600
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

```bash
kubectl apply -f pod-with-config-files.yaml

# Each ConfigMap key becomes a file
kubectl exec config-files-pod -- ls /etc/config/

# Output:
# app.mode
# app.name
# app.properties
# features.json
# log.level
# startup.sh
```

### Lab 7: ConfigMap Auto-Update (Volumes)

**Volumes update automatically (~60s), env vars don't:**

```bash
# Create ConfigMap
kubectl create configmap dynamic-config \
  --from-literal=message="Hello v1"

# Create pod with volume mount
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: dynamic-pod
spec:
  containers:
  - name: app
    image: busybox
    command:
    - sh
    - -c
    - |
      while true; do
        echo "Message: \$(cat /etc/config/message)"
        sleep 5
      done
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: dynamic-config
EOF

# Watch logs
kubectl logs -f dynamic-pod
# Shows: Message: Hello v1

# In another terminal, update ConfigMap
kubectl create configmap dynamic-config \
  --from-literal=message="Hello v2" \
  --dry-run=client -o yaml | kubectl apply -f -

# After ~60 seconds, logs show:
# Message: Hello v2  ✅ Auto-updated!
```

### Lab 8: Environment Variables DON'T Auto-Update

**env-vs-volume.yaml:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-config
data:
  message: "Version 1"
---
apiVersion: v1
kind: Pod
metadata:
  name: env-vs-volume-pod
spec:
  containers:
  - name: app
    image: busybox
    command:
    - sh
    - -c
    - |
      while true; do
        echo "ENV: $MESSAGE_ENV"
        echo "FILE: $(cat /config/message)"
        echo "---"
        sleep 10
      done
    env:
    - name: MESSAGE_ENV
      valueFrom:
        configMapKeyRef:
          name: test-config
          key: message
    volumeMounts:
    - name: config
      mountPath: /config
  volumes:
  - name: config
    configMap:
      name: test-config
```

```bash
kubectl apply -f env-vs-volume.yaml

# Watch logs
kubectl logs -f env-vs-volume-pod
# ENV: Version 1
# FILE: Version 1

# Update ConfigMap
kubectl create configmap test-config \
  --from-literal=message="Version 2" \
  --dry-run=client -o yaml | kubectl apply -f -

# After ~60 seconds:
# ENV: Version 1      ← Still old! ❌
# FILE: Version 2     ← Updated! ✅

# Must restart pod for env var update
kubectl delete pod env-vs-volume-pod
kubectl apply -f env-vs-volume.yaml
```

### Lab 9: TLS Secret for Ingress

```bash
# Generate self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=myapp.local/O=myapp"

# Create TLS secret
kubectl create secret tls myapp-tls-secret \
  --cert=tls.crt \
  --key=tls.key

# View secret
kubectl describe secret myapp-tls-secret

# Type: kubernetes.io/tls
# Data:
#   tls.crt: 1234 bytes
#   tls.key: 5678 bytes
```

**Use in Ingress:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - myapp.local
    secretName: myapp-tls-secret
  rules:
  - host: myapp.local
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

### Lab 10: Docker Registry Secret

```bash
# Create Docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=myusername \
  --docker-password=mypassword \
  --docker-email=myemail@example.com

# View secret type
kubectl get secret regcred -o yaml
# Type: kubernetes.io/dockerconfigjson
```

**Use in Pod:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-image-pod
spec:
  containers:
  - name: app
    image: myusername/private-image:latest
  imagePullSecrets:
  - name: regcred
```

### Lab 11: Immutable ConfigMap

**immutable-configmap.yaml:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: immutable-config
data:
  version: "1.0.0"
  release: "stable"
immutable: true  # Cannot be modified!
```

```bash
kubectl apply -f immutable-configmap.yaml

# Try to update
kubectl create configmap immutable-config \
  --from-literal=version="2.0.0" \
  --dry-run=client -o yaml | kubectl apply -f -

# Error: field is immutable

# Must delete and recreate
kubectl delete configmap immutable-config
kubectl create configmap immutable-config \
  --from-literal=version="2.0.0"
```

### Lab 12: SubPath Mounting

Mount a single file without replacing entire directory:

**pod-with-subpath.yaml:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  default.conf: |
    server {
      listen 80;
      location / {
        return 200 "Custom config!";
      }
    }
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-subpath
spec:
  containers:
  - name: nginx
    image: nginx:1.26
    ports:
    - containerPort: 80
    volumeMounts:
    - name: config
      mountPath: /etc/nginx/conf.d/default.conf
      subPath: default.conf  # Mount only this file
  volumes:
  - name: config
    configMap:
      name: nginx-config
```

```bash
kubectl apply -f pod-with-subpath.yaml

# Without subPath: entire /etc/nginx/conf.d/ replaced
# With subPath: only /etc/nginx/conf.d/default.conf replaced

# Other files in directory still exist
kubectl exec nginx-subpath -- ls /etc/nginx/conf.d/
```

**⚠️ Warning:** SubPath files do NOT auto-update!

### Lab 13: Secret with File Permissions

**secret-with-permissions.yaml:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-perms
stringData:
  sensitive-file: "very secret data"
---
apiVersion: v1
kind: Pod
metadata:
  name: secret-perms-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "ls -la /etc/secrets/ && sleep 3600"]
    volumeMounts:
    - name: secret
      mountPath: /etc/secrets
  volumes:
  - name: secret
    secret:
      secretName: secret-perms
      defaultMode: 0400  # Read-only for owner
```

```bash
kubectl apply -f secret-with-permissions.yaml

# Check permissions
kubectl exec secret-perms-pod -- ls -la /etc/secrets/

# Output:
# -r-------- 1 root root 17 sensitive-file

# Only owner can read, nobody can write/execute
```

### Lab 14: ConfigMap from Directory

```bash
# Create directory with multiple files
mkdir config-dir

cat > config-dir/app.conf <<EOF
port=8080
host=0.0.0.0
EOF

cat > config-dir/database.conf <<EOF
url=postgresql://localhost/mydb
pool_size=10
EOF

# Create ConfigMap from directory
kubectl create configmap app-configs --from-file=config-dir/

# Each file becomes a key
kubectl describe configmap app-configs

# Cleanup
rm -rf config-dir
```

---

## Security Best Practices

### 1. RBAC for Secrets

Limit who can access Secrets:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
  resourceNames: ["specific-secret"]  # Limit to specific secrets
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-secrets
subjects:
- kind: ServiceAccount
  name: my-app
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### 2. Encryption at Rest

Enable encryption for Secrets in etcd:

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: <base64-32-byte-key>
    - identity: {}
```

Configure API server:
```bash
--encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

### 3. External Secret Management

#### HashiCorp Vault

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vault-pod
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "myapp"
    vault.hashicorp.com/agent-inject-secret-db: "secret/data/database"
spec:
  serviceAccountName: vault
  containers:
  - name: app
    image: myapp:latest
```

#### External Secrets Operator (AWS/Azure/GCP)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  secretStoreRef:
    name: aws-secrets
  target:
    name: db-secret
  data:
  - secretKey: password
    remoteRef:
      key: prod/db/password
```

### 4. Sealed Secrets (Bitnami)

Encrypt secrets for Git storage:

```bash
# Install sealed-secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Install kubeseal CLI
brew install kubeseal

# Seal a secret
kubectl create secret generic mysecret \
  --from-literal=password=mysecret123 \
  --dry-run=client -o yaml | \
  kubeseal -o yaml > mysealedsecret.yaml

# Safe to commit to Git!
git add mysealedsecret.yaml
git commit -m "Add sealed secret"

# Apply sealed secret
kubectl apply -f mysealedsecret.yaml

# Controller decrypts and creates real Secret
kubectl get secrets
```

### 5. Never Commit Secrets to Git

**.gitignore:**
```bash
# .gitignore
*.secret.yaml
secrets/
*.key
*.pem
*.crt
.env
credentials/
```

### 6. Audit Secret Access

Enable audit logging:

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]
```

Configure API server:
```bash
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
--audit-log-path=/var/log/kubernetes/audit.log
```

### 7. Security Checklist

```
□ Use Secrets for sensitive data (not ConfigMaps)
□ Enable RBAC to restrict Secret access
□ Enable encryption at rest
□ Use external secret manager in production
□ Never commit secrets to Git
□ Use sealed-secrets for GitOps
□ Rotate secrets regularly
□ Audit secret access
□ Mount secrets as volumes (not env vars) when possible
□ Set readOnly: true on secret volumes
□ Use specific secret permissions (defaultMode)
□ Never log secret values
□ Use immutable secrets when possible
□ Version your secrets (secret-v1, secret-v2)
```

---

## Common Commands

### ConfigMap Commands

```bash
# Create
kubectl create configmap <name> --from-literal=key=value
kubectl create configmap <name> --from-file=file.txt
kubectl create configmap <name> --from-file=dir/
kubectl apply -f configmap.yaml

# View
kubectl get configmaps
kubectl get cm  # Short form
kubectl describe cm <name>
kubectl get cm <name> -o yaml
kubectl get cm <name> -o json

# Edit
kubectl edit cm <name>

# Delete
kubectl delete cm <name>

# Export
kubectl get cm <name> -o yaml > configmap-backup.yaml
```

### Secret Commands

```bash
# Create
kubectl create secret generic <name> --from-literal=key=value
kubectl create secret generic <name> --from-file=file.txt
kubectl create secret tls <name> --cert=cert.crt --key=key.key
kubectl create secret docker-registry <name> \
  --docker-server=... \
  --docker-username=... \
  --docker-password=...
kubectl apply -f secret.yaml

# View
kubectl get secrets
kubectl describe secret <name>  # Shows keys, hides values
kubectl get secret <name> -o yaml  # Shows base64 values

# Decode specific key
kubectl get secret <name> -o jsonpath='{.data.password}' | base64 -d

# Decode all keys
kubectl get secret <name> -o json | jq '.data | map_values(@base64d)'

# Edit
kubectl edit secret <name>

# Delete
kubectl delete secret <name>

# Export (be careful!)
kubectl get secret <name> -o yaml > secret-backup.yaml
```

### Useful Combinations

```bash
# List all ConfigMaps and Secrets
kubectl get cm,secrets

# Find pods using a specific ConfigMap
kubectl get pods -o json | \
  jq -r '.items[] | select(.spec.volumes[]?.configMap.name=="<cm-name>") | .metadata.name'

# Find pods using a specific Secret
kubectl get pods -o json | \
  jq -r '.items[] | select(.spec.volumes[]?.secret.secretName=="<secret-name>") | .metadata.name'

# Base64 encode/decode
echo -n "password" | base64
echo "cGFzc3dvcmQ=" | base64 -d

# Create secret from stdin
kubectl create secret generic mysecret --from-file=password=/dev/stdin
# Type password and press Ctrl+D

# Update ConfigMap from file
kubectl create configmap app-config \
  --from-file=config.yaml \
  --dry-run=client -o yaml | kubectl apply -f -
```

---

## Key Takeaways

### Core Concepts

✅ **ConfigMaps** store non-sensitive configuration  
✅ **Secrets** store sensitive data (base64, NOT encrypted)  
✅ **Three consumption methods**: env vars, args, volumes  
✅ **Volume mounts auto-update** (~60s delay)  
✅ **Env vars don't auto-update** (need pod restart)  
✅ **Base64 ≠ encryption** - use RBAC and encryption at rest  
✅ **Immutable configs** prevent accidental changes  
✅ **External secret managers** for production security  

### ConfigMap vs Secret Decision Tree

```
Is the data sensitive?
├─ NO → Use ConfigMap
│      Examples: API endpoints, feature flags, settings
│
└─ YES → Use Secret
       Examples: Passwords, API keys, certificates, tokens
```

### Update Propagation

| Method | Auto-Update? | Delay | Restart Needed? |
|--------|--------------|-------|-----------------|
| **Environment Variables** | ❌ No | N/A | ✅ Yes |
| **Command Args** | ❌ No | N/A | ✅ Yes |
| **Volume Mounts** | ✅ Yes | ~60 seconds | ❌ No* |

*Unless using subPath

### Best Practices Summary

**Configuration Management:**
1. ✅ Use ConfigMaps for non-sensitive data
2. ✅ Use Secrets for sensitive data
3. ✅ Version your configs (app-config-v1, v2, v3)
4. ✅ Use immutable configs for stability
5. ✅ Prefer volume mounts over env vars (auto-update)
6. ✅ Document your configuration strategy

**Security:**
1. ✅ Never hardcode credentials in images
2. ✅ Enable RBAC for Secret access
3. ✅ Enable encryption at rest
4. ✅ Use external secret managers (Vault, AWS Secrets Manager)
5. ✅ Never commit secrets to Git
6. ✅ Use sealed-secrets for GitOps
7. ✅ Rotate secrets regularly
8. ✅ Audit secret access
9. ✅ Mount secrets as volumes with readOnly: true
10. ✅ Set appropriate file permissions (defaultMode)

**Operations:**
1. ✅ Test config changes in dev first
2. ✅ Use rolling updates for config changes
3. ✅ Monitor application after config updates
4. ✅ Keep backups of configs
5. ✅ Use descriptive names (db-config, not config1)

---

## Homework

### Task 1: Complete Application Stack

Deploy a 3-tier application with proper configuration management:

**Requirements:**
- **Frontend:**
  - ConfigMap: API endpoint, build version, feature flags
  - Use as both env vars and volume
- **Backend:**
  - ConfigMap: Application settings, timeouts, pool sizes
  - Secret: Database password, API keys
  - Use both consumption methods
- **Database:**
  - Secret: Root password, user password
  - ConfigMap: Database configuration file

**Starter Template:**
```yaml
# Database Secret
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  root-password: rootpass123
  user-password: userpass123
  username: appuser
---
# Backend ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
data:
  app.properties: |
    server.port=8080
    database.pool.min=5
    database.pool.max=20
    timeout.connection=30s
---
# Backend Secret
apiVersion: v1
kind: Secret
metadata:
  name: backend-secret
type: Opaque
stringData:
  api-key: sk-prod-abc123xyz789
  jwt-secret: supersecretjwtkey
---
# Backend Deployment
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
        image: myapp/backend:latest
        env:
        - name: DB_HOST
          value: postgres-service
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: user-password
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: backend-secret
              key: api-key
        volumeMounts:
        - name: config
          mountPath: /etc/config
        - name: secrets
          mountPath: /etc/secrets
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: backend-config
      - name: secrets
        secret:
          secretName: backend-secret
```

### Task 2: Blue-Green Deployment with Configs

Implement blue-green deployment using versioned ConfigMaps:

**Requirements:**
1. Create `app-config-blue` (version 1.0)
2. Create `app-config-green` (version 2.0)
3. Deploy blue version with blue config
4. Deploy green version with green config
5. Switch Service selector to green
6. Verify application uses new config

**Template:**
```yaml
# Blue ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-blue
data:
  version: "1.0.0"
  color: "blue"
  message: "This is the BLUE version"
---
# Green ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-green
data:
  version: "2.0.0"
  color: "green"
  message: "This is the GREEN version"
---
# Blue Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args:
        - -text=$(MESSAGE)
        env:
        - name: MESSAGE
          valueFrom:
            configMapKeyRef:
              name: app-config-blue
              key: message
---
# Green Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args:
        - -text=$(MESSAGE)
        env:
        - name: MESSAGE
          valueFrom:
            configMapKeyRef:
              name: app-config-green
              key: message
---
# Service (switch selector to change version)
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: myapp
    version: blue  # Change to 'green' to switch
  ports:
  - port: 80
    targetPort: 5678
```

**Testing:**
```bash
# Deploy both versions
kubectl apply -f blue-green.yaml

# Test blue version
kubectl run test --rm -it --image=curlimages/curl -- curl app-service
# Output: This is the BLUE version

# Switch to green
kubectl patch service app-service -p '{"spec":{"selector":{"version":"green"}}}'

# Test again
kubectl run test --rm -it --image=curlimages/curl -- curl app-service
# Output: This is the GREEN version
```

### Task 3: Secret Rotation (Zero Downtime)

Implement zero-downtime secret rotation:

**Steps:**
1. Create app with initial secret
2. Create new secret with updated credentials
3. Update deployment to use new secret
4. Rolling update ensures zero downtime
5. Verify app uses new credentials
6. Delete old secret

**Example:**
```bash
# 1. Create initial secret
kubectl create secret generic db-creds-v1 \
  --from-literal=password=oldpassword

# 2. Deploy app
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: busybox
        command: ["sh", "-c", "while true; do echo Password: \$DB_PASS; sleep 10; done"]
        env:
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: db-creds-v1
              key: password
EOF

# 3. Create new secret
kubectl create secret generic db-creds-v2 \
  --from-literal=password=newpassword

# 4. Update deployment to use new secret
kubectl set env deployment/myapp --from=secret/db-creds-v2

# 5. Watch rolling update
kubectl rollout status deployment/myapp

# 6. Verify new password
kubectl logs -l app=myapp --tail=1

# 7. Delete old secret
kubectl delete secret db-creds-v1
```

### Task 4: Sealed Secrets Implementation

**Requirements:**
1. Install sealed-secrets controller
2. Create regular secret
3. Seal the secret
4. Commit sealed secret to Git
5. Deploy and verify

**Steps:**
```bash
# 1. Install controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# 2. Install kubeseal CLI
# Mac: brew install kubeseal
# Linux: Download from GitHub releases

# 3. Create and seal secret
kubectl create secret generic mysecret \
  --from-literal=username=admin \
  --from-literal=password=secretpass \
  --dry-run=client -o yaml | \
  kubeseal -o yaml > sealed-secret.yaml

# 4. Safe to commit
cat sealed-secret.yaml
# Shows encrypted data

# 5. Apply sealed secret
kubectl apply -f sealed-secret.yaml

# 6. Controller creates real secret
kubectl get secret mysecret
kubectl get secret mysecret -o jsonpath='{.data.password}' | base64 -d
# Output: secretpass

# 7. Test in pod
kubectl run test --rm -it --image=busybox --restart=Never -- sh
# In pod:
# echo $PASSWORD (if secret mounted as env)
```

### Task 5: External Secrets Research

Research and document ONE external secret manager:

**Choose one:**
- HashiCorp Vault
- AWS Secrets Manager
- Azure Key Vault
- Google Cloud Secret Manager

**Document:**
1. What problem does it solve?
2. How does it integrate with Kubernetes?
3. Basic setup steps
4. Advantages over built-in Secrets
5. Cost considerations
6. Use cases

**Example: HashiCorp Vault**
```markdown
# HashiCorp Vault Integration

## Problem
Kubernetes Secrets are base64 encoded, not encrypted.
Need centralized secret management across multiple clusters.

## Integration
Uses Vault Agent Injector to inject secrets into pods.

## Setup
1. Install Vault
2. Configure Kubernetes auth
3. Store secrets in Vault
4. Annotate pods to inject secrets

## Advantages
- True encryption
- Secret rotation
- Audit logging
- Dynamic secrets
- Multi-cluster support

## Cost
- Open source (free)
- Enterprise features require license

## Use Cases
- Database credentials
- API keys
- TLS certificates
- Cloud provider credentials
```

### Task 6: Config Monitoring Dashboard

Create a simple dashboard showing all ConfigMaps and Secrets:

```bash
# List all ConfigMaps
echo "=== ConfigMaps ==="
kubectl get cm -A -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,KEYS:.data

# List all Secrets
echo "=== Secrets ==="
kubectl get secrets -A -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,TYPE:.type

# Find pods using specific ConfigMap
echo "=== Pods using ConfigMap: app-config ==="
kubectl get pods -o json | \
  jq -r '.items[] | select(.spec.volumes[]?.configMap.name=="app-config") | "\(.metadata.namespace)/\(.metadata.name)"'

# Create a simple monitoring script
cat > config-monitor.sh <<'EOF'
#!/bin/bash
echo "=== Configuration Status Report ==="
echo ""
echo "ConfigMaps: $(kubectl get cm --all-namespaces --no-headers | wc -l)"
echo "Secrets: $(kubectl get secrets --all-namespaces --no-headers | wc -l)"
echo ""
echo "=== Recent ConfigMap Changes ==="
kubectl get cm -A --sort-by=.metadata.creationTimestamp | tail -5
echo ""
echo "=== Recent Secret Changes ==="
kubectl get secrets -A --sort-by=.metadata.creationTimestamp | tail -5
EOF

chmod +x config-monitor.sh
./config-monitor.sh
```

---

## Troubleshooting

### Issue 1: Secret/ConfigMap Not Found

**Symptoms:**
```bash
kubectl get pods
# NAME      READY   STATUS              RESTARTS   AGE
# my-pod    0/1     CreateContainerConfigError   0          10s
```

**Debug:**
```bash
# Check pod events
kubectl describe pod my-pod

# Look for:
# Error: configmap "app-config" not found
# Error: secret "db-secret" not found

# Verify resource exists
kubectl get configmap app-config
kubectl get secret db-secret

# Check namespace
kubectl get configmap app-config -n <namespace>

# Check spelling in pod YAML
kubectl get pod my-pod -o yaml | grep -A 5 "configMap\|secret"
```

**Solution:**
```bash
# Create missing ConfigMap/Secret
kubectl create configmap app-config --from-literal=key=value
kubectl create secret generic db-secret --from-literal=password=pass

# Or check if it's in different namespace
kubectl get cm,secrets -A | grep app-config
```

### Issue 2: Can't Decode Secret

**Symptoms:**
- Need to see secret value for debugging

**Solution:**
```bash
# Method 1: Decode specific key
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
echo ""  # Add newline

# Method 2: Decode all keys
kubectl get secret db-secret -o json | jq -r '.data | to_entries[] | "\(.key): \(.value | @base64d)"'

# Method 3: Use kubectl plugin
kubectl get secret db-secret -o go-template='{{range $k,$v := .data}}{{printf "%s: " $k}}{{if not $v}}{{$v}}{{else}}{{$v | base64decode}}{{end}}{{"\n"}}{{end}}'

# Method 4: Create a debug pod
kubectl run secret-reader --rm -it --image=busybox --restart=Never -- sh
# In pod, if secret mounted:
cat /etc/secrets/password
```

### Issue 3: ConfigMap Changes Not Reflecting

**Symptoms:**
- Updated ConfigMap but pod shows old values

**Debug:**
```bash
# Check if ConfigMap updated
kubectl get configmap app-config -o yaml

# Check how ConfigMap is consumed
kubectl get pod my-pod -o yaml | grep -A 10 "configMap"

# If using env vars:
kubectl describe pod my-pod | grep -A 5 "Environment"
```

**Solution:**
```bash
# For environment variables: Must restart pod
kubectl delete pod my-pod
# Or if Deployment:
kubectl rollout restart deployment/myapp

# For volume mounts: Wait 60 seconds
sleep 60
kubectl exec my-pod -- cat /etc/config/key

# Force immediate update (delete and recreate pod)
kubectl delete pod my-pod
```

### Issue 4: Base64 Encoding Issues

**Symptoms:**
- Secret values have extra characters or don't decode properly

**Problem:**
```bash
# Wrong: includes newline
echo "password" | base64
# Output: cGFzc3dvcmQK  ← Extra characters!

echo "cGFzc3dvcmQK" | base64 -d
# Output: password
#                  ← Extra newline!
```

**Solution:**
```bash
# Correct: use -n flag to prevent newline
echo -n "password" | base64
# Output: cGFzc3dvcmQ=

echo "cGFzc3dvcmQ=" | base64 -d
# Output: password (no newline)

# Or use printf
printf "password" | base64
```

### Issue 5: Pod Can't Mount Secret

**Symptoms:**
```bash
kubectl describe pod my-pod
# Events:
# Warning  FailedMount  Unable to mount volumes for pod
# secret "db-secret" not found
```

**Debug:**
```bash
# 1. Check if secret exists in same namespace
kubectl get secret db-secret -n <pod-namespace>

# 2. Check secret name spelling
kubectl get secrets | grep db

# 3. Check pod YAML
kubectl get pod my-pod -o yaml | grep -A 5 "secret:"

# 4. Verify secret has required keys
kubectl describe secret db-secret
```

**Solution:**
```bash
# Create secret in correct namespace
kubectl create secret generic db-secret \
  --from-literal=password=pass \
  -n <pod-namespace>

# Or reference correct secret name
kubectl edit pod my-pod
# Fix secretName: db-secret
```

### Issue 6: Permission Denied Reading Secret File

**Symptoms:**
```bash
kubectl exec my-pod -- cat /etc/secrets/password
# cat: can't open '/etc/secrets/password': Permission denied
```

**Debug:**
```bash
# Check file permissions
kubectl exec my-pod -- ls -la /etc/secrets/

# Check pod security context
kubectl get pod my-pod -o yaml | grep -A 10 securityContext
```

**Solution:**
```yaml
# Adjust secret defaultMode
volumes:
- name: secret-volume
  secret:
    secretName: db-secret
    defaultMode: 0644  # Readable by all

# Or run container as root (not recommended)
securityContext:
  runAsUser: 0
```

### Issue 7: ConfigMap Too Large

**Symptoms:**
```bash
kubectl create configmap large-config --from-file=large-file.txt
# Error: ConfigMap "large-config" is invalid: data: Too long: must have at most 1048576 bytes
```

**Solution:**
```bash
# Check size
ls -lh large-file.txt

# Option 1: Split into multiple ConfigMaps
split -b 900k large-file.txt chunk_
kubectl create configmap config-part1 --from-file=chunk_aa
kubectl create configmap config-part2 --from-file=chunk_ab

# Option 2: Use PersistentVolume instead
# Option 3: Store in external storage (S3, etc)
# Option 4: Compress the data
gzip large-file.txt
kubectl create configmap compressed-config --from-file=large-file.txt.gz
```

### Debugging Checklist

```bash
# Quick debugging checklist:

□ ConfigMap/Secret exists? → kubectl get cm,secrets
□ Correct namespace? → kubectl get cm,secrets -n <namespace>
□ Correct name in pod YAML? → kubectl get pod <n> -o yaml | grep configMap
□ Keys exist in ConfigMap/Secret? → kubectl describe cm <n>
□ Pod restarted after env var change? → kubectl delete pod <n>
□ Waited 60s for volume mount update? → sleep 60
□ Correct base64 encoding? → echo -n "value" | base64
□ Permissions correct? → kubectl exec <pod> -- ls -la /etc/secrets
□ Check pod events? → kubectl describe pod <n> | grep Events -A 10
□ Check logs? → kubectl logs <pod>
```

---

## Clean Up

```bash
# Delete all ConfigMaps
kubectl delete configmap --all

# Delete all Secrets (careful!)
kubectl delete secret --all

# Delete specific resources
kubectl delete configmap app-config db-config
kubectl delete secret db-secret api-secret myapp-tls-secret

# Delete test pods
kubectl delete pod secret-env-pod secret-volume-pod config-files-pod

# Delete sealed-secrets controller (if installed)
kubectl delete -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Verify cleanup
kubectl get cm,secrets,pods
```

---

## Additional Resources

### Official Documentation
- [Kubernetes ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Encryption at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
- [Managing Secrets](https://kubernetes.io/docs/tasks/configmap-secret/)

### External Secret Management
- [HashiCorp Vault](https://www.vaultproject.io/docs/platform/k8s)
- [External Secrets Operator](https://external-secrets.io/)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/)
- [Azure Key Vault](https://azure.microsoft.com/en-us/products/key-vault/)
- [Google Secret Manager](https://cloud.google.com/secret-manager)

### Tools
- [kubeseal](https://github.com/bitnami-labs/sealed-secrets) - Seal secrets for Git
- [ksd](https://github.com/mfuentesg/ksd) - Kubernetes Secret Decoder
- [kubectl-view-secret](https://github.com/elsesiy/kubectl-view-secret) - View secrets easily
- [k8sec](https://github.com/dtan4/k8sec) - Secret management tool

### Security Resources
- [Kubernetes Security Best Practices](https://kubernetes.io/docs/concepts/security/security-checklist/)
- [RBAC Documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)

---

## Quick Reference

### ConfigMap Templates

**Simple ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "postgres://localhost/mydb"
  log_level: "info"
```

**ConfigMap with Files:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.properties: |
    server.port=8080
    db.pool.size=10
  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://backend;
      }
    }
```

### Secret Templates

**Opaque Secret (stringData):**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  username: admin
  password: secretpass123
```

**TLS Secret:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-cert>
  tls.key: <base64-key>
```

**Docker Registry Secret:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: regcred
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-docker-config>
```

### Pod Consumption Templates

**Environment Variables:**
```yaml
env:
- name: DB_HOST
  valueFrom:
    configMapKeyRef:
      name: db-config
      key: host
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password

# Or all at once:
envFrom:
- configMapRef:
    name: app-config
- secretRef:
    name: app-secret
```

**Volume Mounts:**
```yaml
volumeMounts:
- name: config
  mountPath: /etc/config
- name: secrets
  mountPath: /etc/secrets
  readOnly: true

volumes:
- name: config
  configMap:
    name: app-config
- name: secrets
  secret:
    secretName: app-secret
    defaultMode: 0400
```

---

## Summary

**Congratulations!** You've completed Lesson 7 on ConfigMaps & Secrets.

### What You Learned

✅ **ConfigMaps** for application configuration  
✅ **Secrets** for sensitive data  
✅ **Multiple creation methods** - literal, file, YAML  
✅ **Three consumption methods** - env vars, args, volumes  
✅ **Update propagation** - volumes auto-update, env vars don't  
✅ **Security best practices** - RBAC, encryption, external managers  
✅ **Secret types** - Opaque, TLS, Docker, SSH  
✅ **Immutable configs** for stability  
✅ **External secret management** - Vault, sealed-secrets  

### Skills Acquired

- Create and manage ConfigMaps and Secrets
- Consume configs in multiple ways
- Implement config versioning and rotation
- Secure sensitive data properly
- Use external secret managers
- Troubleshoot config issues
- Implement zero-downtime updates

### Production Readiness

You can now:
- Design configuration management strategy
- Secure sensitive data properly
- Implement config versioning
- Rotate secrets without downtime
- Use external secret management
- Follow security best practices
- Debug configuration issues

---

## What's Next?

**Upcoming Lessons:**
- **Lesson 8**: StatefulSets (Deep Dive) - stateful applications
- **Lesson 9**: DaemonSets, Jobs & CronJobs - specialized workloads
- **Lesson 10**: RBAC & Security - access control
- **Lesson 11**: Resource Management - requests, limits, quotas
- **Lesson 12**: Health Checks & Probes - application reliability

**Recommended Practice:**
1. Complete all homework tasks
2. Implement secret rotation in a real app
3. Set up sealed-secrets for GitOps
4. Research external secret managers
5. Practice troubleshooting config issues

---

**Happy Learning! 🚀**

Ready to move to Lesson 8 (StatefulSets)?