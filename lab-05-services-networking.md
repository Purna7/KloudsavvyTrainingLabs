# Lab 05: Services & Networking

## Lab Overview

| Item | Detail |
|------|--------|
| **Duration** | 100 minutes |
| **Level** | Intermediate |
| **Prerequisites** | Modules 01-04 completed |
| **Objectives** | Service types, Ingress, DNS discovery, Network Policies |

---

## Part 1: Service Types (25 minutes)

### Step 1.1: Deploy Backend and Frontend

Create `microservices.yaml`:
```yaml
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
        image: gcr.io/google-samples/hello-app:1.0
        ports:
        - containerPort: 8080
        env:
        - name: SERVICE_NAME
          value: "backend"
---
# Frontend Deployment
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
      - name: frontend
        image: nginx:1.25
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f microservices.yaml
kubectl get pods -l 'app in (backend,frontend)'
```

### Step 1.2: ClusterIP Service

Create `svc-clusterip.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-clusterip
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
```

```bash
kubectl apply -f svc-clusterip.yaml

# View the service
kubectl get svc backend-clusterip

# Test from within the cluster
kubectl run test-client --image=busybox:1.36 --rm -it --restart=Never -- \
  wget -qO- http://backend-clusterip

# Verify DNS resolution
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup backend-clusterip
```

### Step 1.3: NodePort Service

Create `svc-nodeport.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-nodeport
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

```bash
kubectl apply -f svc-nodeport.yaml

# Access via minikube
minikube service frontend-nodeport --url
# Open the URL in browser

# Or directly
curl $(minikube ip):30080
```

### Step 1.4: Examine Endpoints

```bash
# Services track pod IPs via Endpoints
kubectl get endpoints backend-clusterip
kubectl get endpoints frontend-nodeport

# Scale backend and watch endpoints change
kubectl scale deployment backend --replicas=5
kubectl get endpoints backend-clusterip
# More IPs now!

kubectl scale deployment backend --replicas=3
```

> **Checkpoint 1:** Both service types working, DNS resolution confirmed.

---

## Part 2: Service Discovery with DNS (15 minutes)

### Step 2.1: Create a Debug Pod

```bash
kubectl run dns-explorer --image=busybox:1.36 -it --restart=Never -- /bin/sh
```

Inside the pod:
```bash
# DNS Config
cat /etc/resolv.conf
# Shows: nameserver 10.96.0.10 (CoreDNS)
#        search default.svc.cluster.local svc.cluster.local cluster.local

# Resolve service names
nslookup backend-clusterip
nslookup backend-clusterip.default
nslookup backend-clusterip.default.svc.cluster.local

# Access the backend via DNS
wget -qO- http://backend-clusterip:80

# Check kube-system services
nslookup kubernetes.default
nslookup kube-dns.kube-system

exit
```

```bash
kubectl delete pod dns-explorer
```

### Step 2.2: Cross-Namespace DNS

```bash
# Create a namespace
kubectl create namespace staging

# Deploy app in staging
kubectl create deployment staging-app --image=nginx:1.25 -n staging
kubectl expose deployment staging-app --port=80 -n staging

# Access from default namespace
kubectl run cross-ns-test --image=busybox:1.36 --rm -it --restart=Never -- \
  wget -qO- http://staging-app.staging

# DNS: <service>.<namespace>
```

> **Checkpoint 2:** DNS service discovery working across namespaces.

---

## Part 3: Ingress Controller (25 minutes)

### Step 3.1: Enable Ingress

```bash
# Enable nginx ingress on minikube
minikube addons enable ingress

# Wait for the ingress controller
kubectl get pods -n ingress-nginx -w
# Wait until running
```

### Step 3.2: Create Ingress Rules

Create `ingress.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-nodeport
            port:
              number: 80
      - path: /api(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: backend-clusterip
            port:
              number: 80
```

```bash
kubectl apply -f ingress.yaml
kubectl get ingress

# Get minikube IP
echo "$(minikube ip) myapp.local" | sudo tee -a /etc/hosts
# On Windows: Add to C:\Windows\System32\drivers\etc\hosts

# Test
curl http://myapp.local        # Frontend (nginx)
curl http://myapp.local/api    # Backend (hello-app)
```

### Step 3.3: Multiple Hosts Ingress

Create `ingress-multi.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: frontend.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-nodeport
            port:
              number: 80
  - host: backend.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-clusterip
            port:
              number: 80
```

```bash
kubectl apply -f ingress-multi.yaml

# Add hosts entries
MINIKUBE_IP=$(minikube ip)
echo "$MINIKUBE_IP frontend.local" | sudo tee -a /etc/hosts
echo "$MINIKUBE_IP backend.local" | sudo tee -a /etc/hosts

# Test
curl http://frontend.local
curl http://backend.local
```

> **Checkpoint 3:** Ingress routing working with path and host-based rules.

---

## Part 4: Network Policies (20 minutes)

### Step 4.1: Deploy a Test Environment

Create `netpol-setup.yaml`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: netpol-lab
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: netpol-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
      role: frontend
  template:
    metadata:
      labels:
        app: web
        role: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: netpol-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
      role: backend
  template:
    metadata:
      labels:
        app: api
        role: backend
    spec:
      containers:
      - name: api
        image: gcr.io/google-samples/hello-app:1.0
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: web-svc
  namespace: netpol-lab
spec:
  selector:
    app: web
  ports:
  - port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: api-svc
  namespace: netpol-lab
spec:
  selector:
    app: api
  ports:
  - port: 8080
```

```bash
kubectl apply -f netpol-setup.yaml

# Verify connectivity (should work — no policies yet)
kubectl run test -n netpol-lab --image=busybox:1.36 --rm -it --restart=Never -- \
  wget -qO- --timeout=3 http://web-svc
kubectl run test -n netpol-lab --image=busybox:1.36 --rm -it --restart=Never -- \
  wget -qO- --timeout=3 http://api-svc:8080
```

### Step 4.2: Default Deny All

Create `netpol-deny.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: netpol-lab
spec:
  podSelector: {}         # Selects ALL pods
  policyTypes:
  - Ingress
  - Egress
```

```bash
kubectl apply -f netpol-deny.yaml

# Test — should FAIL (timeout)
kubectl run test -n netpol-lab --image=busybox:1.36 --rm -it --restart=Never -- \
  wget -qO- --timeout=3 http://web-svc
# wget: download timed out
```

### Step 4.3: Allow Specific Traffic

Create `netpol-allow.yaml`:
```yaml
# Allow frontend → backend traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: netpol-lab
spec:
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - port: 8080
---
# Allow DNS resolution for all pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: netpol-lab
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to: []
    ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
---
# Allow frontend egress to backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-egress
  namespace: netpol-lab
spec:
  podSelector:
    matchLabels:
      role: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: backend
    ports:
    - port: 8080
```

```bash
kubectl apply -f netpol-allow.yaml

# Test from frontend pod → backend: should WORK
kubectl exec -n netpol-lab $(kubectl get pod -n netpol-lab -l role=frontend -o name) -- \
  wget -qO- --timeout=3 http://api-svc:8080

# Test from random pod → backend: should FAIL
kubectl run attacker -n netpol-lab --image=busybox:1.36 --rm -it --restart=Never -- \
  wget -qO- --timeout=3 http://api-svc:8080
```

> **Checkpoint 4:** Network isolation enforced — only frontend can reach backend.

---

## Part 5: Debugging Networking (15 minutes)

### Step 5.1: Common Debug Commands

```bash
# Check service endpoints
kubectl get endpoints -n netpol-lab

# Check if service selector matches pods
kubectl get pods -n netpol-lab --show-labels

# DNS debugging
kubectl run dns-debug -n netpol-lab --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup api-svc.netpol-lab.svc.cluster.local

# Check network policies
kubectl get networkpolicies -n netpol-lab
kubectl describe networkpolicy -n netpol-lab

# Check kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=10
```

### Step 5.2: Common Issues Checklist

| Issue | Check |
|-------|-------|
| Service not reachable | `kubectl get endpoints` — are pods listed? |
| DNS not resolving | `nslookup` from debug pod, check CoreDNS logs |
| Connection timeout | Check NetworkPolicy, pod labels |
| Intermittent failures | Check readiness probes, endpoint changes |

---

## Cleanup

```bash
kubectl delete namespace netpol-lab staging
kubectl delete -f microservices.yaml
kubectl delete -f svc-clusterip.yaml -f svc-nodeport.yaml
kubectl delete ingress app-ingress multi-host-ingress
# Remove /etc/hosts entries manually
```

---

## Challenge Exercises

### Challenge 1: DockerComposeDemo on Kubernetes
Convert your `docker-compose.yml` networking to Kubernetes:
1. Create ClusterIP service for backend (port 3000)
2. Create ClusterIP service for database (port 5432)
3. Create Ingress that routes `/api/*` to backend, `/` to frontend
4. Verify frontend can reach backend, backend can reach database

### Challenge 2: Network Policy Audit
Design network policies for a 3-tier architecture:
- Frontend: accepts traffic from ingress controller only
- Backend: accepts traffic from frontend only
- Database: accepts traffic from backend only
- All: can resolve DNS

---

## Lab Review Checklist

- [ ] Created ClusterIP and NodePort services
- [ ] Verified DNS service discovery within and across namespaces
- [ ] Set up Ingress with path and host-based routing
- [ ] Implemented default-deny NetworkPolicy
- [ ] Created allow policies for specific traffic flows
- [ ] Debugged networking issues

## Next Lab
→ [Lab 06: Storage & Persistence](../Module-06-Storage/lab-06-storage.md)
