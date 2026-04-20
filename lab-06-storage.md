# Lab 06: Storage & Persistence

## Lab Overview

| Item | Detail |
|------|--------|
| **Duration** | 90 minutes |
| **Level** | Intermediate |
| **Prerequisites** | Modules 01-05 completed |
| **Objectives** | Volumes, PV/PVC, StorageClasses, persistent PostgreSQL |

---

## Part 1: Volume Types (20 minutes)

### Step 1.1: emptyDir — Shared Temporary Storage

Create `vol-emptydir.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
  - name: writer
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      i=0
      while true; do
        echo "$(date) - Entry $i" >> /data/log.txt
        i=$((i+1))
        sleep 5
      done
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: reader
    image: busybox:1.36
    command: ["/bin/sh", "-c", "tail -f /data/log.txt"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
      readOnly: true
  volumes:
  - name: shared-data
    emptyDir: {}
```

```bash
kubectl apply -f vol-emptydir.yaml

# Read logs from reader container
kubectl logs emptydir-demo -c reader -f
# You'll see log entries written by the writer container

# Verify shared storage
kubectl exec emptydir-demo -c writer -- ls -la /data/
kubectl exec emptydir-demo -c reader -- cat /data/log.txt
```

### Step 1.2: hostPath — Node-Local Storage

Create `vol-hostpath.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Written at $(date)" > /host-data/test.txt
      cat /host-data/test.txt
      sleep 3600
    volumeMounts:
    - name: host-vol
      mountPath: /host-data
  volumes:
  - name: host-vol
    hostPath:
      path: /tmp/k8s-lab-data
      type: DirectoryOrCreate
```

```bash
kubectl apply -f vol-hostpath.yaml

# Verify data on the node
minikube ssh -- cat /tmp/k8s-lab-data/test.txt

# Delete and recreate the pod — data persists!
kubectl delete pod hostpath-demo
kubectl apply -f vol-hostpath.yaml
minikube ssh -- cat /tmp/k8s-lab-data/test.txt
```

> **Checkpoint 1:** emptyDir shares data between containers; hostPath persists on the node.

---

## Part 2: PersistentVolumes & Claims (25 minutes)

### Step 2.1: Manual PV Provisioning

Create `pv-manual.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-pv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /tmp/k8s-pv-data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: manual-pvc
spec:
  storageClassName: manual
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

```bash
kubectl apply -f pv-manual.yaml

# Check PV and PVC status
kubectl get pv
kubectl get pvc

# PVC should show "Bound" status
kubectl describe pvc manual-pvc
```

### Step 2.2: Use PVC in a Pod

Create `pod-with-pvc.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Hello from PVC at $(date)" >> /persistent/data.txt
      echo "Contents:"
      cat /persistent/data.txt
      sleep 3600
    volumeMounts:
    - name: persistent-storage
      mountPath: /persistent
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: manual-pvc
```

```bash
kubectl apply -f pod-with-pvc.yaml
kubectl logs pvc-demo

# Delete and recreate — data survives!
kubectl delete pod pvc-demo
kubectl apply -f pod-with-pvc.yaml
kubectl logs pvc-demo
# Should show BOTH "Hello" entries
```

### Step 2.3: Dynamic Provisioning

```bash
# Check available StorageClasses
kubectl get storageclass

# minikube has a default "standard" StorageClass
kubectl describe storageclass standard
```

Create `pvc-dynamic.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  # Uses default StorageClass since none specified
```

```bash
kubectl apply -f pvc-dynamic.yaml
kubectl get pvc dynamic-pvc
# STATUS should be "Bound"

# A PV was automatically created!
kubectl get pv
```

> **Checkpoint 2:** Manual and dynamic PV provisioning working.

---

## Part 3: PostgreSQL with Persistent Storage (25 minutes)

### Step 3.1: Create the Secret

```bash
kubectl create secret generic postgres-secret \
  --from-literal=username=demouser \
  --from-literal=password=demopass123
```

### Step 3.2: Deploy PostgreSQL

Create `postgres-persistent.yaml`:
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
      storage: 1Gi
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
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: demodb
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            cpu: 250m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "demouser", "-d", "demodb"]
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          exec:
            command: ["pg_isready", "-U", "demouser", "-d", "demodb"]
          initialDelaySeconds: 30
          periodSeconds: 10
      volumes:
      - name: postgres-data
        persistentVolumeClaim:
          claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

```bash
kubectl apply -f postgres-persistent.yaml

# Wait for PostgreSQL to be ready
kubectl get pods -l app=postgres -w

# Connect and create data
kubectl exec -it $(kubectl get pod -l app=postgres -o name) -- \
  psql -U demouser -d demodb -c "
    CREATE TABLE IF NOT EXISTS messages (
      id SERIAL PRIMARY KEY,
      content TEXT,
      created_at TIMESTAMP DEFAULT NOW()
    );
    INSERT INTO messages (content) VALUES ('Hello from Kubernetes!');
    INSERT INTO messages (content) VALUES ('Data survives restarts!');
    SELECT * FROM messages;
  "
```

### Step 3.3: Test Persistence

```bash
# Delete the PostgreSQL pod (NOT the deployment or PVC)
kubectl delete pod $(kubectl get pod -l app=postgres -o name)

# Deployment recreates it
kubectl get pods -l app=postgres -w

# Query data — it should still be there!
kubectl exec -it $(kubectl get pod -l app=postgres -o name) -- \
  psql -U demouser -d demodb -c "SELECT * FROM messages;"
```

**Data survives pod deletion!** The PVC maintains the storage.

> **Checkpoint 3:** PostgreSQL deployed with persistent storage that survives pod restarts.

---

## Part 4: Storage Exploration (15 minutes)

### Step 4.1: Inspect Storage

```bash
# List all PV and PVC
kubectl get pv -o wide
kubectl get pvc -o wide

# Detailed PVC info
kubectl describe pvc postgres-pvc

# Check storage utilization
kubectl exec -it $(kubectl get pod -l app=postgres -o name) -- \
  df -h /var/lib/postgresql/data
```

### Step 4.2: Volume Expansion

```bash
# Check if StorageClass allows expansion
kubectl get storageclass standard -o yaml | grep allowVolumeExpansion

# If supported, expand the PVC
kubectl patch pvc postgres-pvc -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'
kubectl get pvc postgres-pvc
```

---

## Cleanup

```bash
kubectl delete -f postgres-persistent.yaml
kubectl delete secret postgres-secret
kubectl delete -f pv-manual.yaml
kubectl delete -f pvc-dynamic.yaml
kubectl delete pod emptydir-demo hostpath-demo pvc-demo
minikube ssh -- rm -rf /tmp/k8s-lab-data /tmp/k8s-pv-data
```

---

## Challenge Exercises

### Challenge 1: Multi-Pod Shared Storage
Deploy two pods that share the same PVC and can both read/write data.

### Challenge 2: Database Backup CronJob
Create a Kubernetes CronJob that:
1. Mounts the same PVC as PostgreSQL
2. Runs `pg_dump` every hour
3. Stores backups in a separate PVC

### Challenge 3: DockerComposeDemo Database Migration
Recreate the PostgreSQL setup from `docker-compose.yml`:
1. Use the `init.sql` script as a ConfigMap
2. Mount it at `/docker-entrypoint-initdb.d/`
3. Verify tables are created on first start

---

## Lab Review Checklist

- [ ] Created emptyDir and hostPath volumes
- [ ] Provisioned PV manually and dynamically
- [ ] Deployed PostgreSQL with persistent storage
- [ ] Verified data survives pod deletion
- [ ] Explored PVC status and capacity

## Next Lab
→ [Lab 07: ConfigMaps & Secrets](../Module-07-ConfigMaps-Secrets/lab-07-configmaps-secrets.md)
