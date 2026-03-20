# Lab 02: Exploring Kubernetes Architecture

## Lab Overview

| Item | Detail |
|------|--------|
| **Duration** | 90 minutes |
| **Level** | Beginner |
| **Prerequisites** | Lab 01 completed, minikube running |
| **Objectives** | Inspect control plane components, trace pod lifecycle, understand etcd |

---

## Part 1: Inspect Control Plane Components (25 minutes)

### Step 1.1: List All System Components

```bash
# See all control plane pods
kubectl get pods -n kube-system -o wide

# List with more details
kubectl get pods -n kube-system -o custom-columns=\
'NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName,IP:.status.podIP'
```

**Expected components:**
- `etcd-minikube`
- `kube-apiserver-minikube`
- `kube-controller-manager-minikube`
- `kube-scheduler-minikube`
- `coredns-*`
- `kube-proxy-*`

### Step 1.2: Examine the API Server

```bash
# Describe the API server pod
kubectl describe pod -n kube-system kube-apiserver-minikube

# Look for:
# - Command line arguments (--etcd-servers, --service-cluster-ip-range)
# - Resource requests/limits
# - Liveness probe configuration
# - Volumes mounted (certificates)
```

**Questions to answer:**
1. What port is the API server listening on?
2. What is the service cluster IP range?
3. Where are the TLS certificates stored?

### Step 1.3: Examine etcd

```bash
# Describe etcd
kubectl describe pod -n kube-system etcd-minikube

# Check etcd data directory
kubectl exec -n kube-system etcd-minikube -- ls /var/lib/minikube/etcd/

# Check etcd member list
kubectl exec -n kube-system etcd-minikube -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/minikube/certs/etcd/ca.crt \
  --cert=/var/lib/minikube/certs/etcd/server.crt \
  --key=/var/lib/minikube/certs/etcd/server.key \
  member list -w table
```

### Step 1.4: Examine the Scheduler

```bash
# Describe scheduler
kubectl describe pod -n kube-system kube-scheduler-minikube

# Check scheduler logs
kubectl logs -n kube-system kube-scheduler-minikube --tail=20
```

### Step 1.5: Examine the Controller Manager

```bash
# Describe controller manager
kubectl describe pod -n kube-system kube-controller-manager-minikube

# Check controller manager logs
kubectl logs -n kube-system kube-controller-manager-minikube --tail=20
```

> **Checkpoint 1:** Document the command-line arguments for each control plane component. Note which flags control behavior like leader election, bind addresses, and certificate paths.

---

## Part 2: Trace a Pod Creation (25 minutes)

### Step 2.1: Set Up Event Watching

Open **two terminal windows**:

**Terminal 1 — Watch events in real-time:**
```bash
kubectl get events -w --sort-by='.lastTimestamp'
```

**Terminal 2 — Watch pods:**
```bash
kubectl get pods -w
```

### Step 2.2: Create a Pod and Observe

In a **third terminal**, create a pod:

```bash
kubectl run trace-pod --image=nginx:1.25 --restart=Never
```

**Observe Terminal 1 (Events).** You should see events in this order:

```
TYPE     REASON      OBJECT           MESSAGE
Normal   Scheduled   pod/trace-pod    Successfully assigned default/trace-pod to minikube
Normal   Pulling     pod/trace-pod    Pulling image "nginx:1.25"
Normal   Pulled      pod/trace-pod    Successfully pulled image "nginx:1.25"
Normal   Created     pod/trace-pod    Created container trace-pod
Normal   Started     pod/trace-pod    Started container trace-pod
```

**Map each event to the architecture:**
| Event | Component Responsible |
|-------|----------------------|
| Scheduled | kube-scheduler |
| Pulling | kubelet → containerd |
| Pulled | kubelet → containerd |
| Created | kubelet |
| Started | kubelet |

### Step 2.3: Describe the Pod in Detail

```bash
kubectl describe pod trace-pod
```

**Examine these sections:**
- **Node:** Which node was it scheduled to?
- **IP:** What IP was assigned?
- **Containers:** Image, state, ready status
- **Conditions:** Initialized, Ready, ContainersReady, PodScheduled
- **Events:** Full lifecycle events
- **QoS Class:** BestEffort (no resource requests/limits set)

### Step 2.4: Check the API Server Audit

```bash
# View what the API server processed
kubectl logs -n kube-system kube-apiserver-minikube --tail=50 | grep trace-pod
```

> **Checkpoint 2:** You should be able to explain each step of pod creation and which component handled it.

---

## Part 3: Explore kubelet and kube-proxy (20 minutes)

### Step 3.1: Access the Minikube Node

```bash
# SSH into the minikube node
minikube ssh
```

### Step 3.2: Inspect kubelet

```bash
# Inside minikube node:

# Check kubelet process
ps aux | grep kubelet

# Check kubelet configuration
cat /var/lib/kubelet/config.yaml

# Check kubelet logs
journalctl -u kubelet --no-pager --lines=30

# See what kubelet knows about pods
ls /var/lib/kubelet/pods/
```

### Step 3.3: Inspect Container Runtime

```bash
# Still inside minikube:

# List running containers using crictl (CRI CLI)
sudo crictl ps

# List images
sudo crictl images

# Get container details
sudo crictl inspect <container-id>
```

### Step 3.4: Inspect kube-proxy

```bash
# Still inside minikube:

# Check iptables rules created by kube-proxy
sudo iptables -t nat -L KUBE-SERVICES -n | head -20

# Check kube-proxy mode
ps aux | grep kube-proxy

# Exit minikube SSH
exit
```

**Back on your host:**
```bash
# Check kube-proxy logs
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=20
```

> **Checkpoint 3:** You've seen how kubelet manages pods, containerd runs containers, and kube-proxy creates iptables rules.

---

## Part 4: etcd Data Exploration (15 minutes)

### Step 4.1: Read Data from etcd

```bash
# List keys in etcd (shows all stored resources)
kubectl exec -n kube-system etcd-minikube -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/minikube/certs/etcd/ca.crt \
  --cert=/var/lib/minikube/certs/etcd/server.crt \
  --key=/var/lib/minikube/certs/etcd/server.key \
  get / --prefix --keys-only | head -30
```

**You'll see keys like:**
```
/registry/pods/default/trace-pod
/registry/services/specs/default/kubernetes
/registry/namespaces/default
/registry/serviceaccounts/default/default
```

### Step 4.2: Read a Specific Pod from etcd

```bash
kubectl exec -n kube-system etcd-minikube -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/minikube/certs/etcd/ca.crt \
  --cert=/var/lib/minikube/certs/etcd/server.crt \
  --key=/var/lib/minikube/certs/etcd/server.key \
  get /registry/pods/default/trace-pod
```

> **Note:** The output is Protocol Buffers encoded — not human-readable, but this shows you that etcd stores ALL Kubernetes state.

### Step 4.3: Watch etcd Changes

```bash
# In one terminal, watch etcd for changes
kubectl exec -n kube-system etcd-minikube -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/minikube/certs/etcd/ca.crt \
  --cert=/var/lib/minikube/certs/etcd/server.crt \
  --key=/var/lib/minikube/certs/etcd/server.key \
  watch / --prefix

# In another terminal, create a configmap
kubectl create configmap test-config --from-literal=key=value
```

You'll see etcd changes in real-time!

---

## Part 5: API Server Exploration (15 minutes)

### Step 5.1: Direct API Access

```bash
# Start kubectl proxy (creates a local proxy to API server)
kubectl proxy --port=8001 &

# List all available API groups
curl -s http://localhost:8001/apis | python3 -m json.tool | head -40

# Get pods via REST API
curl -s http://localhost:8001/api/v1/namespaces/default/pods | python3 -m json.tool | head -30

# Get a specific pod
curl -s http://localhost:8001/api/v1/namespaces/default/pods/trace-pod | python3 -m json.tool

# List all available resources
curl -s http://localhost:8001/api/v1 | python3 -m json.tool | grep '"name"' | head -20

# Stop the proxy
kill %1
```

### Step 5.2: API Discovery

```bash
# List all API resources and their group/version
kubectl api-resources -o wide

# List all API versions
kubectl api-versions

# Explain a resource structure
kubectl explain pod
kubectl explain pod.spec.containers
kubectl explain deployment.spec.strategy
```

---

## Cleanup

```bash
kubectl delete pod trace-pod
kubectl delete configmap test-config
```

---

## Challenge Exercises

### Challenge 1: Component Failure Simulation
1. Scale the CoreDNS deployment to 0: `kubectl scale deployment coredns -n kube-system --replicas=0`
2. Try creating a pod that needs DNS — what happens?
3. Restore: `kubectl scale deployment coredns -n kube-system --replicas=2`

### Challenge 2: Static Pod Discovery
1. Find where static pod manifests are stored on the minikube node
2. Hint: SSH into minikube and check `/etc/kubernetes/manifests/`
3. Create a static pod by placing a manifest in that directory
4. Observe it appear in `kubectl get pods -n kube-system`

### Challenge 3: API Server Resource Map
Write a script that:
1. Lists all API resources
2. Counts resources per API group
3. Identifies which resources are namespaced vs cluster-scoped

---

## Lab Review Checklist

- [ ] Identified all control plane components and their args
- [ ] Traced pod creation through events and logs
- [ ] Explored kubelet, containerd, and kube-proxy on the node
- [ ] Read data directly from etcd
- [ ] Accessed the API server via REST API
- [ ] Used `kubectl explain` to explore resource schemas

---

## Key Takeaways

1. The **API Server** is the central hub — everything communicates through it
2. **etcd** stores everything — losing it means losing the cluster
3. The **Scheduler** only decides placement, **kubelet** does the actual work
4. **Controller Manager** is what makes Kubernetes "declarative" — constant reconciliation
5. Understanding these components is essential for **CKA certification** and production debugging

## Next Lab
→ [Lab 03: Pods & Workloads](../Module-03-Pods-Workloads/lab-03-pods-and-workloads.md)
