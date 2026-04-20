# Lab 08: RBAC & Namespaces

## Lab Overview

| Item | Detail |
|------|--------|
| **Duration** | 90 minutes |
| **Level** | Intermediate |
| **Prerequisites** | Modules 01-07 completed |
| **Objectives** | Namespaces, RBAC roles, ServiceAccounts, ResourceQuotas |

---

## Part 1: Namespace Management (20 minutes)

### Step 1.1: Create Namespaces

```bash
kubectl create namespace production
kubectl create namespace staging
kubectl create namespace development

kubectl get namespaces
```

### Step 1.2: Deploy to Specific Namespaces

```bash
# Deploy to production
kubectl create deployment web-prod --image=nginx:1.25 -n production --replicas=3
kubectl expose deployment web-prod --port=80 -n production

# Deploy to staging
kubectl create deployment web-staging --image=nginx:1.25 -n staging --replicas=1
kubectl expose deployment web-staging --port=80 -n staging

# View resources per namespace
kubectl get all -n production
kubectl get all -n staging
kubectl get pods --all-namespaces
```

### Step 1.3: Set Default Namespace

```bash
# Create a context with different default namespace
kubectl config set-context --current --namespace=production

# Now kubectl commands default to 'production'
kubectl get pods  # Shows production pods

# Switch back to default
kubectl config set-context --current --namespace=default
```

> **Checkpoint 1:** Namespaces created with isolated deployments.

---

## Part 2: RBAC — Roles and Bindings (30 minutes)

### Step 2.1: Create a ServiceAccount (Simulating a User)

```bash
kubectl create serviceaccount dev-user -n development
kubectl create serviceaccount viewer -n production
```

### Step 2.2: Create a Read-Only Role

Create `rbac-roles.yaml`:
```yaml
# Read-only role for developers
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-viewer
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "services", "endpoints"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch"]
---
# Full access for developers in development namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-full
  namespace: development
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["*"]
  verbs: ["*"]
---
# Bind viewer role to viewer service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: viewer-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: viewer
  namespace: production
roleRef:
  kind: Role
  name: pod-viewer
  apiGroup: rbac.authorization.k8s.io
---
# Bind developer role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-full-binding
  namespace: development
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: development
roleRef:
  kind: Role
  name: developer-full
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f rbac-roles.yaml
```

### Step 2.3: Test RBAC Permissions

```bash
# Test what the viewer can do in production
kubectl auth can-i get pods -n production --as system:serviceaccount:production:viewer
# yes

kubectl auth can-i delete pods -n production --as system:serviceaccount:production:viewer
# no

kubectl auth can-i create deployments -n production --as system:serviceaccount:production:viewer
# no

# Test what developer can do in development
kubectl auth can-i create deployments -n development --as system:serviceaccount:development:dev-user
# yes

kubectl auth can-i delete pods -n development --as system:serviceaccount:development:dev-user
# yes

# Developer should NOT have access to production
kubectl auth can-i get pods -n production --as system:serviceaccount:development:dev-user
# no
```

### Step 2.4: ClusterRole for Cross-Namespace Read

Create `rbac-cluster.yaml`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: global-pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "namespaces"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: security-audit-binding
subjects:
- kind: ServiceAccount
  name: viewer
  namespace: production
roleRef:
  kind: ClusterRole
  name: global-pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f rbac-cluster.yaml

# Now viewer can read pods in ANY namespace
kubectl auth can-i list pods -n staging --as system:serviceaccount:production:viewer
# yes

kubectl auth can-i list pods -n development --as system:serviceaccount:production:viewer
# yes
```

> **Checkpoint 2:** RBAC roles working with proper access controls.

---

## Part 3: ResourceQuotas & LimitRanges (20 minutes)

### Step 3.1: Apply ResourceQuota

Create `resource-quota.yaml`:
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    pods: "5"
    requests.cpu: "2"
    requests.memory: 2Gi
    limits.cpu: "4"
    limits.memory: 4Gi
    services: "3"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: development
spec:
  limits:
  - default:
      cpu: 200m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: "1"
      memory: 1Gi
    min:
      cpu: 50m
      memory: 64Mi
    type: Container
```

```bash
kubectl apply -f resource-quota.yaml

# Check quota
kubectl get resourcequota -n development
kubectl describe resourcequota dev-quota -n development
```

### Step 3.2: Test Quota Enforcement

```bash
# Deploy within quota
kubectl create deployment quota-test --image=nginx:1.25 -n development --replicas=3
kubectl get pods -n development

# Try to exceed pod quota
kubectl scale deployment quota-test --replicas=10 -n development
kubectl get pods -n development
kubectl describe resourcequota dev-quota -n development
# Only 5 pods allowed!

# Check events for quota violations
kubectl get events -n development --sort-by='.lastTimestamp' | tail -5
```

### Step 3.3: Test LimitRange Defaults

```bash
# Create a pod without specifying resources
kubectl run no-limits --image=busybox:1.36 -n development --command -- sleep 3600

# Check that defaults were applied
kubectl get pod no-limits -n development -o yaml | grep -A 4 resources
# cpu: 200m and memory: 256Mi were automatically set!
```

> **Checkpoint 3:** ResourceQuotas prevent overconsumption, LimitRanges set defaults.

---

## Part 4: ServiceAccount for Application Identity (15 minutes)

### Step 4.1: Pod with Custom ServiceAccount

Create `sa-pod.yaml`:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: development
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
  namespace: development
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-configmap-access
  namespace: development
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: development
roleRef:
  kind: Role
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Pod
metadata:
  name: sa-demo
  namespace: development
spec:
  serviceAccountName: app-sa
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "=== Testing ServiceAccount permissions ==="
      echo "Can list configmaps:"
      kubectl get configmaps 2>&1
      echo ""
      echo "Can list secrets (should fail):"
      kubectl get secrets 2>&1
      echo ""
      echo "Can list pods (should fail):"
      kubectl get pods 2>&1
      sleep 3600
```

```bash
kubectl apply -f sa-pod.yaml
sleep 10
kubectl logs sa-demo -n development
```

---

## Cleanup

```bash
kubectl delete namespace production staging development
kubectl delete clusterrole global-pod-reader
kubectl delete clusterrolebinding security-audit-binding
```

---

## Challenge Exercises

### Challenge 1: Team Onboarding Script
Write a script that creates:
1. A namespace for a new team
2. ResourceQuota and LimitRange
3. Read-only role + team RoleBinding
4. Deploy role + lead developer RoleBinding

### Challenge 2: Audit Existing Permissions
Use `kubectl auth can-i --list` to audit what the default service account can do in each namespace.

---

## Lab Review Checklist

- [ ] Created namespaces for environment isolation
- [ ] Created Roles and RoleBindings for access control
- [ ] Created ClusterRole for cross-namespace access
- [ ] Tested RBAC with `kubectl auth can-i`
- [ ] Applied ResourceQuotas and verified enforcement
- [ ] Verified LimitRange default injection
- [ ] Used custom ServiceAccount for pod identity

## Next Lab
→ [Lab 09: Helm & Package Management](../Module-09-Helm/lab-09-helm.md)
