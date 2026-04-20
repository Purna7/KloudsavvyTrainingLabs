# Lab 07: ConfigMaps & Secrets

## Lab Overview

| Item | Detail |
|------|--------|
| **Duration** | 75 minutes |
| **Level** | Intermediate |
| **Prerequisites** | Modules 01-06 completed |
| **Objectives** | Create/use ConfigMaps and Secrets, mount as env vars and files |

---

## Part 1: ConfigMaps (25 minutes)

### Step 1.1: Create ConfigMaps Multiple Ways

```bash
# From literals
kubectl create configmap app-settings \
  --from-literal=APP_COLOR=blue \
  --from-literal=APP_MODE=production \
  --from-literal=MAX_RETRIES=3

# View it
kubectl get configmap app-settings -o yaml
```

Create `configmap.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-files-config
data:
  # Simple key-value
  database.host: "postgres-svc"
  database.port: "5432"
  
  # Multi-line config file
  app.properties: |
    server.port=8080
    server.timeout=30
    logging.level=INFO
    feature.darkmode=true
    feature.newui=false
  
  # JSON config file
  config.json: |
    {
      "database": {
        "host": "postgres-svc",
        "port": 5432,
        "name": "demodb",
        "pool_size": 10
      },
      "cache": {
        "enabled": true,
        "ttl_seconds": 300
      }
    }
```

```bash
kubectl apply -f configmap.yaml
kubectl describe configmap app-files-config
```

### Step 1.2: Use ConfigMap as Environment Variables

Create `pod-configmap-env.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-env-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "=== All env vars from configmap ==="
      echo "APP_COLOR: $APP_COLOR"
      echo "APP_MODE: $APP_MODE"
      echo "MAX_RETRIES: $MAX_RETRIES"
      echo ""
      echo "=== Specific env var ==="
      echo "DB_HOST: $DB_HOST"
      echo "DB_PORT: $DB_PORT"
      sleep 3600
    envFrom:
    - configMapRef:
        name: app-settings
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-files-config
          key: database.host
    - name: DB_PORT
      valueFrom:
        configMapKeyRef:
          name: app-files-config
          key: database.port
```

```bash
kubectl apply -f pod-configmap-env.yaml
kubectl logs configmap-env-demo
```

### Step 1.3: Use ConfigMap as Volume Files

Create `pod-configmap-vol.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-vol-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "=== Config files mounted ==="
      ls -la /etc/config/
      echo ""
      echo "=== app.properties ==="
      cat /etc/config/app.properties
      echo ""
      echo "=== config.json ==="
      cat /etc/config/config.json
      sleep 3600
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
      readOnly: true
  volumes:
  - name: config-vol
    configMap:
      name: app-files-config
      items:
      - key: app.properties
        path: app.properties
      - key: config.json
        path: config.json
```

```bash
kubectl apply -f pod-configmap-vol.yaml
kubectl logs configmap-vol-demo
```

### Step 1.4: Hot-Reload ConfigMap (Volume-Mounted)

```bash
# Update the configmap
kubectl edit configmap app-files-config
# Change logging.level=INFO to logging.level=DEBUG

# Wait ~30 seconds, then check the pod
kubectl exec configmap-vol-demo -- cat /etc/config/app.properties
# The file is updated without pod restart!
```

> **Note:** Volume-mounted ConfigMaps auto-update. Environment variables do NOT.

> **Checkpoint 1:** ConfigMaps working as env vars and files, hot-reload verified.

---

## Part 2: Secrets (25 minutes)

### Step 2.1: Create Secrets

```bash
# Create from literals
kubectl create secret generic db-credentials \
  --from-literal=username=demouser \
  --from-literal=password='P@ssw0rd!123'

# View the secret (base64 encoded)
kubectl get secret db-credentials -o yaml

# Decode a value
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d
```

Create `secret.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:                    # stringData accepts plain text
  api-key: "sk-1234567890abcdef"
  jwt-secret: "super-secret-jwt-signing-key"
  database-url: "postgresql://demouser:P@ssw0rd!123@postgres-svc:5432/demodb"
```

```bash
kubectl apply -f secret.yaml
kubectl get secret app-secrets -o yaml
# Note: values are now base64-encoded in the stored version
```

### Step 2.2: Use Secrets in Pods

Create `pod-secrets.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "=== Secrets as Env Vars ==="
      echo "DB_USER: $DB_USER"
      echo "DB_PASS: [REDACTED for security]"
      echo "API_KEY: ${API_KEY:0:5}*****"
      echo ""
      echo "=== Secrets as Files ==="
      echo "JWT Secret file exists: $(test -f /etc/secrets/jwt-secret && echo 'yes' || echo 'no')"
      ls -la /etc/secrets/
      sleep 3600
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: api-key
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-vol
    secret:
      secretName: app-secrets
      defaultMode: 0400        # Read-only for owner
```

```bash
kubectl apply -f pod-secrets.yaml
kubectl logs secret-demo
```

### Step 2.3: TLS Secret

```bash
# Generate self-signed cert (for lab purposes)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=myapp.local/O=LabOrg"

# Create TLS secret
kubectl create secret tls app-tls-secret \
  --cert=tls.crt --key=tls.key

kubectl get secret app-tls-secret -o yaml
```

### Step 2.4: Docker Registry Secret

```bash
# Create registry credential (for pulling private images)
kubectl create secret docker-registry my-registry \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=user@example.com

# Use in a pod
# spec:
#   imagePullSecrets:
#   - name: my-registry
```

> **Checkpoint 2:** Secrets created and used as env vars and files.

---

## Part 3: Real-World Application Config (20 minutes)

### Step 3.1: Full-Stack Configuration

Create `fullstack-config.yaml`:
```yaml
# Application Configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
data:
  NODE_ENV: "production"
  PORT: "3000"
  DB_HOST: "postgres-svc"
  DB_PORT: "5432"
  DB_NAME: "demodb"
  LOG_LEVEL: "info"
  CORS_ORIGIN: "http://frontend-svc"
---
# Database Credentials
apiVersion: v1
kind: Secret
metadata:
  name: backend-secrets
type: Opaque
stringData:
  DB_USER: "demouser"
  DB_PASSWORD: "SecurePass123!"
  SESSION_SECRET: "random-session-secret-key"
---
# Nginx Configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  default.conf: |
    upstream backend {
        server backend-svc:3000;
    }
    server {
        listen 80;
        server_name localhost;

        location / {
            root /usr/share/nginx/html;
            index index.html;
            try_files $uri $uri/ /index.html;
        }

        location /api/ {
            proxy_pass http://backend/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
---
# Frontend with mounted nginx config
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
        image: nginx:1.25
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-conf
          mountPath: /etc/nginx/conf.d
      volumes:
      - name: nginx-conf
        configMap:
          name: nginx-config
---
# Backend with config + secrets
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
      - name: app
        image: gcr.io/google-samples/hello-app:1.0
        ports:
        - containerPort: 8080
        envFrom:
        - configMapRef:
            name: backend-config
        - secretRef:
            name: backend-secrets
```

```bash
kubectl apply -f fullstack-config.yaml

# Verify env vars in backend pod
kubectl exec $(kubectl get pod -l app=backend -o name | head -1) -- env | sort

# Verify nginx config in frontend
kubectl exec $(kubectl get pod -l app=frontend -o name | head -1) -- \
  cat /etc/nginx/conf.d/default.conf
```

---

## Cleanup

```bash
kubectl delete -f fullstack-config.yaml
kubectl delete configmap app-settings app-files-config nginx-config backend-config
kubectl delete secret db-credentials app-secrets backend-secrets app-tls-secret my-registry
kubectl delete pod configmap-env-demo configmap-vol-demo secret-demo
rm -f tls.crt tls.key
```

---

## Challenge Exercises

### Challenge 1: Environment-Specific Configs
Create separate ConfigMaps for dev/staging/prod and a script that switches between them.

### Challenge 2: Sealed Secrets
Research and implement Bitnami Sealed Secrets for encrypted secret storage in Git.

### Challenge 3: DockerComposeDemo Migration
Convert all configuration from your `docker-compose.yml` environment variables to ConfigMaps and Secrets.

---

## Lab Review Checklist

- [ ] Created ConfigMaps from literals, files, and YAML
- [ ] Used ConfigMaps as env vars and volume mounts
- [ ] Verified hot-reload of volume-mounted ConfigMaps
- [ ] Created Secrets (Opaque, TLS, Docker registry)
- [ ] Used Secrets as env vars and files
- [ ] Configured a full-stack app with externalized config

## Next Lab
→ [Lab 08: RBAC & Namespaces](../Module-08-RBAC-Namespaces/lab-08-rbac-namespaces.md)
