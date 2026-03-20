# Lab 01: Setting Up Your First Kubernetes Cluster

## Lab Overview

| Item | Detail |
|------|--------|
| **Duration** | 90 minutes |
| **Level** | Beginner |
| **Prerequisites** | Docker installed, terminal access |
| **Objectives** | Install kubectl, set up minikube, deploy first app, explore cluster |

---

## Part 1: Install Prerequisites (20 minutes)

### Step 1.1: Install kubectl

**Windows (PowerShell as Administrator):**
```powershell
choco install kubernetes-cli -y
```

**macOS:**
```bash
brew install kubectl
```

**Linux:**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

**Verify installation:**
```bash
kubectl version --client --output=yaml
```

**Expected Output:**
```
clientVersion:
  gitVersion: "v1.29.x"
  platform: linux/amd64
```

### Step 1.2: Install minikube

**Windows:**
```powershell
choco install minikube -y
```

**macOS:**
```bash
brew install minikube
```

**Linux:**
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64
```

### Step 1.3: Start Your Cluster

```bash
minikube start --cpus=4 --memory=8192 --driver=docker --kubernetes-version=stable
```

**Expected Output:**
```
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

### Step 1.4: Verify the Cluster

```bash
# Check cluster info
kubectl cluster-info

# Check nodes
kubectl get nodes -o wide

# Check system pods
kubectl get pods -n kube-system
```

**Expected Output for `kubectl get nodes`:**
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   1m    v1.29.0
```

> **Checkpoint 1:** You should see a single node in `Ready` status.

---

## Part 2: Explore the Cluster (20 minutes)

### Step 2.1: Understand kubectl Configuration

```bash
# View current context
kubectl config current-context

# View all contexts (you may have multiple clusters later)
kubectl config get-contexts

# View full config
kubectl config view
```

**Discussion:** The kubeconfig file (usually `~/.kube/config`) stores:
- **Clusters** — API server endpoints
- **Users** — Authentication credentials
- **Contexts** — Cluster + user + namespace combinations

### Step 2.2: Explore Namespaces

```bash
# List all namespaces
kubectl get namespaces

# See what's running in kube-system
kubectl get pods -n kube-system

# See ALL resources in ALL namespaces
kubectl get all --all-namespaces
```

**Expected Namespaces:**
```
NAME              STATUS   AGE
default           Active   5m
kube-node-lease   Active   5m
kube-public       Active   5m
kube-system       Active   5m
```

### Step 2.3: Explore the kube-system Namespace

```bash
# List all system components
kubectl get pods -n kube-system -o wide

# Describe a specific component
kubectl describe pod -n kube-system -l component=etcd

# Check the API server
kubectl describe pod -n kube-system -l component=kube-apiserver
```

> **Checkpoint 2:** You should see system pods like `etcd`, `kube-apiserver`, `kube-scheduler`, `kube-controller-manager`, `coredns`, and `kube-proxy`.

---

## Part 3: Deploy Your First Application (25 minutes)

### Step 3.1: Imperative Deployment

```bash
# Create a deployment
kubectl create deployment hello-k8s --image=gcr.io/google-samples/hello-app:1.0

# Watch the pod come up
kubectl get pods -w
```

Press `Ctrl+C` once the pod shows `Running`.

```bash
# Check the deployment
kubectl get deployments
kubectl describe deployment hello-k8s
```

### Step 3.2: Expose the Application

```bash
# Create a NodePort service
kubectl expose deployment hello-k8s --type=NodePort --port=8080

# Get the service URL
minikube service hello-k8s --url
```

**Open the URL in your browser.** You should see:
```
Hello, world!
Version: 1.0.0
Hostname: hello-k8s-xxxxxxxxxx-xxxxx
```

### Step 3.3: Scale the Application

```bash
# Scale to 3 replicas
kubectl scale deployment hello-k8s --replicas=3

# Watch new pods being created
kubectl get pods -w

# After all pods are Running, refresh the browser multiple times
# Notice the Hostname changes — load balancing is working!
```

### Step 3.4: Declarative Deployment

Create a file `hello-k8s.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-k8s-declarative
  labels:
    app: hello-k8s-declarative
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-k8s-declarative
  template:
    metadata:
      labels:
        app: hello-k8s-declarative
    spec:
      containers:
      - name: hello-app
        image: gcr.io/google-samples/hello-app:2.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 100m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: hello-k8s-declarative
spec:
  type: NodePort
  selector:
    app: hello-k8s-declarative
  ports:
  - port: 8080
    targetPort: 8080
```

```bash
# Apply the manifest
kubectl apply -f hello-k8s.yaml

# Check the resources
kubectl get deployment hello-k8s-declarative
kubectl get service hello-k8s-declarative

# Access the app
minikube service hello-k8s-declarative --url
```

> **Checkpoint 3:** You should see "Hello, world! Version: 2.0.0" in your browser.

---

## Part 4: Basic Operations Practice (15 minutes)

### Step 4.1: Viewing Logs

```bash
# Get logs from a specific pod
kubectl logs <pod-name>

# Stream logs in real-time
kubectl logs -f <pod-name>

# Get logs from all pods with a label
kubectl logs -l app=hello-k8s --all-containers
```

### Step 4.2: Executing Commands in Pods

```bash
# Get a shell inside a pod
kubectl exec -it <pod-name> -- /bin/sh

# Inside the container, run:
hostname
env | grep KUBERNETES
cat /etc/resolv.conf
wget -qO- localhost:8080
exit
```

### Step 4.3: Describing Resources (Debugging)

```bash
# Detailed info about a pod
kubectl describe pod <pod-name>

# Look for:
# - Events section (scheduling, pulling image, starting)
# - Conditions (Ready, ContainersReady)
# - Container state (Running, Waiting, Terminated)
```

### Step 4.4: Watching Events

```bash
# View cluster events sorted by time
kubectl get events --sort-by='.lastTimestamp'

# Watch events in real-time
kubectl get events -w
```

---

## Part 5: Self-Healing Demonstration (10 minutes)

### Step 5.1: Kill a Pod and Watch Kubernetes Recover

```bash
# List pods
kubectl get pods -l app=hello-k8s

# Delete one pod (simulate a crash)
kubectl delete pod <one-of-the-pod-names>

# Immediately watch — K8s creates a replacement!
kubectl get pods -l app=hello-k8s -w
```

**What happened?**
- The Deployment controller detected that the **actual state** (2 pods) didn't match the **desired state** (3 pods)
- It immediately created a new pod to restore the desired state
- This is **self-healing** — a core Kubernetes feature

### Step 5.2: Try to Delete All Pods

```bash
# Delete ALL pods for the deployment
kubectl delete pods -l app=hello-k8s

# Watch what happens
kubectl get pods -l app=hello-k8s -w
```

ALL pods are recreated! The Deployment controller won't give up.

> **To truly remove the app, delete the Deployment (the controller), not the pods.**

---

## Part 6: Enable Essential Addons (10 minutes)

### Step 6.1: Kubernetes Dashboard

```bash
# Enable the dashboard
minikube addons enable dashboard

# Launch it
minikube dashboard
```

This opens a web UI where you can:
- View workloads, services, and configs
- See pod logs and exec into containers
- Visualize resource usage

### Step 6.2: Metrics Server

```bash
# Enable metrics
minikube addons enable metrics-server

# Wait 60 seconds for metrics to be collected, then:
kubectl top nodes
kubectl top pods
```

### Step 6.3: Ingress Controller

```bash
# Enable ingress (you'll use this in Module 05)
minikube addons enable ingress

# Verify
kubectl get pods -n ingress-nginx
```

---

## Cleanup

```bash
# Delete all resources we created
kubectl delete deployment hello-k8s
kubectl delete service hello-k8s
kubectl delete -f hello-k8s.yaml

# Verify cleanup
kubectl get all

# (Optional) Stop minikube to save resources
minikube stop

# (Optional) Delete the cluster entirely
minikube delete
```

---

## Challenge Exercises

### Challenge 1: Deploy Your DockerComposeDemo Backend
Deploy the Node.js backend from this project to Kubernetes:
1. Build the image: `docker build -t demo-backend:v1 ./backend`
2. Load it into minikube: `minikube image load demo-backend:v1`
3. Create a deployment using that image
4. Expose it as a NodePort service
5. Test the `/health` endpoint

### Challenge 2: Multiple Contexts
1. Create a second minikube cluster: `minikube start -p cluster2`
2. Switch between contexts with `kubectl config use-context`
3. Deploy different apps to each cluster
4. Delete the second cluster: `minikube delete -p cluster2`

### Challenge 3: Resource Exploration
Write a script that outputs:
- Number of nodes
- Total pods across all namespaces
- All available API resources (`kubectl api-resources`)
- Cluster version

---

## Lab Review Checklist

- [ ] kubectl installed and working
- [ ] minikube cluster running with single node
- [ ] Successfully deployed app imperatively
- [ ] Successfully deployed app declaratively with YAML
- [ ] Can view logs and exec into pods
- [ ] Demonstrated self-healing behavior
- [ ] Dashboard and metrics-server enabled
- [ ] Completed at least one challenge exercise

---

## Key Takeaways

1. **minikube** provides a full Kubernetes cluster locally
2. **kubectl** is your primary tool — learn it well
3. **Declarative YAML** is how real teams manage infrastructure
4. Kubernetes **self-heals** — it constantly reconciles actual state with desired state
5. The **kube-system** namespace contains the cluster's own components

## Next Lab
→ [Lab 02: Exploring Kubernetes Architecture](../Module-02-Architecture/lab-02-architecture-exploration.md)
