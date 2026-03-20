# Lab 04: Deployments & ReplicaSets

## Lab Overview

| Item | Detail |
|------|--------|
| **Duration** | 90 minutes |
| **Level** | Intermediate |
| **Prerequisites** | Modules 01-03 completed |
| **Objectives** | Rolling updates, rollbacks, blue-green, canary deployments |

---

## Part 1: Create and Manage a Deployment (20 minutes)

### Step 1.1: Create the Deployment

Create `deployment-v1.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  annotations:
    kubernetes.io/change-cause: "Initial deployment v1"
spec:
  replicas: 4
  selector:
    matchLabels:
      app: webapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: webapp
        version: v1
    spec:
      containers:
      - name: web
        image: gcr.io/google-samples/hello-app:1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 100m
            memory: 128Mi
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 3
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-svc
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
  - port: 8080
    targetPort: 8080
```

```bash
kubectl apply -f deployment-v1.yaml

# Verify
kubectl get deployment webapp
kubectl get replicaset -l app=webapp
kubectl get pods -l app=webapp -o wide
kubectl get service webapp-svc

# Get URL
minikube service webapp-svc --url
```

### Step 1.2: Explore the Hierarchy

```bash
# See the ReplicaSet created by the Deployment
kubectl get rs -l app=webapp -o wide

# Note the pod template hash in the ReplicaSet name
# webapp-<hash> — this hash changes with each update

# Describe the deployment
kubectl describe deployment webapp
```

> **Checkpoint 1:** 4 pods running, all accessible via the service.

---

## Part 2: Rolling Update (20 minutes)

### Step 2.1: Perform a Rolling Update

Open **Terminal 1** — Watch pods:
```bash
kubectl get pods -l app=webapp -w
```

Open **Terminal 2** — Watch ReplicaSets:
```bash
kubectl get rs -l app=webapp -w
```

**Terminal 3** — Trigger the update:
```bash
kubectl set image deployment/webapp web=gcr.io/google-samples/hello-app:2.0
kubectl annotate deployment/webapp kubernetes.io/change-cause="Update to v2.0" --overwrite
```

**Observe:**
- Terminal 1: New pods created, old pods terminated gradually
- Terminal 2: New ReplicaSet scales up, old scales down

```bash
# Monitor rollout status
kubectl rollout status deployment/webapp

# Verify the update
kubectl describe deployment webapp | grep Image

# Access the app — should show Version: 2.0.0
minikube service webapp-svc --url
```

### Step 2.2: View Rollout History

```bash
kubectl rollout history deployment/webapp

# Output should show:
# REVISION  CHANGE-CAUSE
# 1         Initial deployment v1
# 2         Update to v2.0
```

> **Checkpoint 2:** Successful zero-downtime rolling update.

---

## Part 3: Rollback (15 minutes)

### Step 3.1: Simulate a Bad Deployment

```bash
# Deploy a "bad" version (image that doesn't exist)
kubectl set image deployment/webapp web=gcr.io/google-samples/hello-app:bad-version
kubectl annotate deployment/webapp kubernetes.io/change-cause="BROKEN: bad image tag" --overwrite

# Watch the failure
kubectl rollout status deployment/webapp --timeout=30s

# Check — some pods will be in ImagePullBackOff
kubectl get pods -l app=webapp
```

### Step 3.2: Rollback to Previous Version

```bash
# Quick rollback
kubectl rollout undo deployment/webapp

# Verify recovery
kubectl rollout status deployment/webapp
kubectl get pods -l app=webapp

# Check image
kubectl describe deployment webapp | grep Image
```

### Step 3.3: Rollback to Specific Revision

```bash
# View history
kubectl rollout history deployment/webapp

# Rollback to revision 1 (original v1)
kubectl rollout undo deployment/webapp --to-revision=1

# Verify
kubectl rollout status deployment/webapp
kubectl describe deployment webapp | grep Image
# Should show hello-app:1.0
```

> **Checkpoint 3:** Successfully recovered from a bad deployment using rollback.

---

## Part 4: Blue-Green Deployment (15 minutes)

### Step 4.1: Deploy "Blue" (Current)

Create `blue-green.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp-bg
      version: blue
  template:
    metadata:
      labels:
        app: webapp-bg
        version: blue
    spec:
      containers:
      - name: web
        image: gcr.io/google-samples/hello-app:1.0
        ports:
        - containerPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp-bg
      version: green
  template:
    metadata:
      labels:
        app: webapp-bg
        version: green
    spec:
      containers:
      - name: web
        image: gcr.io/google-samples/hello-app:2.0
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-bg-svc
spec:
  type: NodePort
  selector:
    app: webapp-bg
    version: blue              # Points to BLUE
  ports:
  - port: 8080
    targetPort: 8080
```

```bash
kubectl apply -f blue-green.yaml

# Verify — service points to blue (v1)
minikube service webapp-bg-svc --url
# Shows: Version: 1.0.0
```

### Step 4.2: Switch to Green

```bash
# Instant switch to green
kubectl patch service webapp-bg-svc -p '{"spec":{"selector":{"version":"green"}}}'

# Verify — now shows v2
minikube service webapp-bg-svc --url
# Shows: Version: 2.0.0
```

### Step 4.3: Rollback (Switch Back to Blue)

```bash
kubectl patch service webapp-bg-svc -p '{"spec":{"selector":{"version":"blue"}}}'
# Instant rollback!
```

> **Checkpoint 4:** Blue-green deployment with instant switching.

---

## Part 5: Canary Deployment (10 minutes)

### Step 5.1: Deploy Canary

Create `canary.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-stable
spec:
  replicas: 9                    # 90% of traffic
  selector:
    matchLabels:
      app: webapp-canary
      track: stable
  template:
    metadata:
      labels:
        app: webapp-canary
        track: stable
    spec:
      containers:
      - name: web
        image: gcr.io/google-samples/hello-app:1.0
        ports:
        - containerPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-canary
spec:
  replicas: 1                    # 10% of traffic
  selector:
    matchLabels:
      app: webapp-canary
      track: canary
  template:
    metadata:
      labels:
        app: webapp-canary
        track: canary
    spec:
      containers:
      - name: web
        image: gcr.io/google-samples/hello-app:2.0
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-canary-svc
spec:
  type: NodePort
  selector:
    app: webapp-canary          # Selects BOTH stable and canary
  ports:
  - port: 8080
    targetPort: 8080
```

```bash
kubectl apply -f canary.yaml

# Test — refresh multiple times, ~10% will show v2
URL=$(minikube service webapp-canary-svc --url)
for i in $(seq 1 20); do curl -s $URL; done | sort | uniq -c
```

### Step 5.2: Promote Canary

```bash
# Gradually increase canary, decrease stable
kubectl scale deployment webapp-canary --replicas=5
kubectl scale deployment webapp-stable --replicas=5

# If everything looks good, full promotion
kubectl scale deployment webapp-canary --replicas=10
kubectl scale deployment webapp-stable --replicas=0
```

---

## Cleanup

```bash
kubectl delete deployment webapp webapp-blue webapp-green webapp-stable webapp-canary
kubectl delete service webapp-svc webapp-bg-svc webapp-canary-svc
```

---

## Challenge Exercises

### Challenge 1: Deployment with maxUnavailable
1. Create a deployment with `maxUnavailable: 50%` and `maxSurge: 0`
2. Observe how the update differs from `maxSurge: 1, maxUnavailable: 0`
3. When would you use each strategy?

### Challenge 2: Recreate Strategy
1. Create a deployment with `strategy: Recreate`
2. Update the image and observe — all old pods die before new ones start
3. Note the brief downtime period

### Challenge 3: Automated Canary
Write a shell script that:
1. Deploys a canary at 10%
2. Waits 30 seconds and checks health
3. Promotes to 50% if healthy
4. Waits 30 seconds and checks again
5. Promotes to 100% or rolls back

---

## Lab Review Checklist

- [ ] Created deployment with 4 replicas
- [ ] Performed zero-downtime rolling update
- [ ] Viewed rollout history
- [ ] Rolled back from a bad deployment
- [ ] Implemented blue-green deployment
- [ ] Implemented canary deployment
- [ ] Understood when to use each strategy

## Next Lab
→ [Lab 05: Services & Networking](../Module-05-Services-Networking/lab-05-services-networking.md)
