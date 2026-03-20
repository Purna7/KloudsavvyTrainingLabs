# Lab 03: Pods & Workloads Hands-On

## Lab Overview

| Item | Detail |
|------|--------|
| **Duration** | 90 minutes |
| **Level** | Beginner |
| **Prerequisites** | Lab 01-02 completed, minikube running |
| **Objectives** | Create pods, multi-container pods, init containers, probes, debug issues |

---

## Part 1: Pod Basics (20 minutes)

### Step 1.1: Create a Pod Declaratively

Create `pod-basic.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-server
  labels:
    app: web
    environment: lab
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
      name: http
```

```bash
kubectl apply -f pod-basic.yaml
kubectl get pod web-server -o wide
kubectl describe pod web-server
```

### Step 1.2: Access the Pod

```bash
# Port forward to access locally
kubectl port-forward pod/web-server 8080:80 &

# Test it
curl http://localhost:8080

# Stop port-forward
kill %1
```

### Step 1.3: Explore Inside the Pod

```bash
# Get a shell
kubectl exec -it web-server -- /bin/bash

# Inside the container:
hostname
ip addr
cat /etc/nginx/nginx.conf
curl localhost:80
apt-get update && apt-get install -y procps
ps aux
exit
```

### Step 1.4: Pod with Resource Limits

Create `pod-resources.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
  labels:
    app: resource-demo
spec:
  containers:
  - name: stress-test
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "100M", "--timeout", "60s"]
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 200m
        memory: 256Mi
```

```bash
kubectl apply -f pod-resources.yaml
kubectl get pod resource-demo -o wide

# Watch resource usage (metrics-server must be enabled)
kubectl top pod resource-demo

# Check QoS class
kubectl get pod resource-demo -o jsonpath='{.status.qosClass}'
```

> **Checkpoint 1:** Pod running with defined resource constraints.

---

## Part 2: Multi-Container Pods (25 minutes)

### Step 2.1: Sidecar Pattern — Log Shipping

Create `pod-sidecar.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
  labels:
    app: sidecar-demo
spec:
  containers:
  # Main application - writes logs
  - name: app
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      i=0
      while true; do
        echo "$(date) - Log entry $i: Application processing request" >> /var/log/app.log
        i=$((i+1))
        sleep 3
      done
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log

  # Sidecar - reads and processes logs
  - name: log-reader
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Waiting for log file..."
      while [ ! -f /var/log/app.log ]; do sleep 1; done
      echo "Tailing logs from sidecar:"
      tail -f /var/log/app.log
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log
      readOnly: true

  volumes:
  - name: shared-logs
    emptyDir: {}
```

```bash
kubectl apply -f pod-sidecar.yaml

# Check both containers
kubectl get pod sidecar-demo

# View logs from the app container
kubectl logs sidecar-demo -c app

# View logs from the sidecar
kubectl logs sidecar-demo -c log-reader -f
```

**Key Learning:** Both containers share the same volume. The app writes, the sidecar reads.

### Step 2.2: Ambassador Pattern — Database Proxy

Create `pod-ambassador.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ambassador-demo
spec:
  containers:
  # Main app connects to localhost:6379
  - name: app
    image: redis:7
    command: ["sh", "-c"]
    args:
    - |
      while true; do
        echo "App: Checking connection to ambassador on localhost:6380..."
        redis-cli -p 6380 ping 2>/dev/null || echo "Ambassador not ready yet"
        sleep 5
      done

  # Ambassador - local proxy
  - name: ambassador
    image: redis:7
    command: ["redis-server", "--port", "6380"]
    ports:
    - containerPort: 6380
```

```bash
kubectl apply -f pod-ambassador.yaml
kubectl logs ambassador-demo -c app -f
```

### Step 2.3: Adapter Pattern — Metrics Conversion

Create `pod-adapter.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: adapter-demo
spec:
  containers:
  # App that generates custom metrics
  - name: app
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      while true; do
        # Generate custom metrics in proprietary format
        RAND=$((RANDOM % 100))
        echo "metric:requests_total=$RAND" > /metrics/raw-metrics.txt
        echo "metric:error_count=$((RAND / 10))" >> /metrics/raw-metrics.txt
        echo "metric:latency_ms=$((RAND * 3))" >> /metrics/raw-metrics.txt
        sleep 5
      done
    volumeMounts:
    - name: metrics-vol
      mountPath: /metrics

  # Adapter converts to Prometheus format
  - name: adapter
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      while true; do
        if [ -f /metrics/raw-metrics.txt ]; then
          echo "# Converted Prometheus Metrics:"
          while IFS= read -r line; do
            key=$(echo $line | cut -d= -f1 | sed 's/metric://')
            value=$(echo $line | cut -d= -f2)
            echo "${key} ${value}"
          done < /metrics/raw-metrics.txt
          echo "---"
        fi
        sleep 5
      done
    volumeMounts:
    - name: metrics-vol
      mountPath: /metrics
      readOnly: true

  volumes:
  - name: metrics-vol
    emptyDir: {}
```

```bash
kubectl apply -f pod-adapter.yaml
kubectl logs adapter-demo -c adapter -f
```

> **Checkpoint 2:** You've built all three multi-container patterns.

---

## Part 3: Init Containers (15 minutes)

### Step 3.1: Init Container — Wait for Dependency

Create `pod-init.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  # Init 1: Simulate waiting for a service
  - name: wait-for-service
    image: busybox:1.36
    command: ['sh', '-c', 'echo "Waiting for service..." && sleep 5 && echo "Service ready!"']

  # Init 2: Prepare data
  - name: prepare-data
    image: busybox:1.36
    command: ['sh', '-c']
    args:
    - |
      echo "Preparing application data..."
      echo "<h1>Initialized by init container at $(date)</h1>" > /work-dir/index.html
      echo "Data preparation complete!"
    volumeMounts:
    - name: workdir
      mountPath: /work-dir

  containers:
  - name: web
    image: nginx:1.25
    ports:
    - containerPort: 80
    volumeMounts:
    - name: workdir
      mountPath: /usr/share/nginx/html

  volumes:
  - name: workdir
    emptyDir: {}
```

```bash
kubectl apply -f pod-init.yaml

# Watch init containers run sequentially
kubectl get pod init-demo -w

# Once running, verify the init container's work
kubectl exec init-demo -- cat /usr/share/nginx/html/index.html

# Check init container logs
kubectl logs init-demo -c wait-for-service
kubectl logs init-demo -c prepare-data
```

### Step 3.2: Real-World Init — Database Migration

Create `pod-init-migration.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-migration
spec:
  initContainers:
  - name: check-db-ready
    image: busybox:1.36
    command: ['sh', '-c']
    args:
    - |
      echo "Checking if database is reachable..."
      # In real scenario: until nc -z database-service 5432; do sleep 2; done
      echo "Simulating DB check..."
      sleep 3
      echo "Database is ready!"

  - name: run-migration
    image: busybox:1.36
    command: ['sh', '-c']
    args:
    - |
      echo "Running database migrations..."
      echo "ALTER TABLE users ADD COLUMN IF NOT EXISTS last_login TIMESTAMP;"
      echo "CREATE INDEX IF NOT EXISTS idx_users_email ON users(email);"
      sleep 2
      echo "Migrations completed successfully!"

  containers:
  - name: app
    image: nginx:1.25
    env:
    - name: DB_MIGRATED
      value: "true"
```

```bash
kubectl apply -f pod-init-migration.yaml
kubectl get pod app-with-migration -w
kubectl logs app-with-migration -c run-migration
```

> **Checkpoint 3:** Init containers run sequentially before the main container starts.

---

## Part 4: Health Probes (20 minutes)

### Step 4.1: Liveness Probe — Auto-Recovery

Create `pod-liveness.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      # Create health file
      touch /tmp/healthy
      echo "App started, healthy file created"
      # After 20 seconds, remove the file (simulate failure)
      sleep 20
      rm /tmp/healthy
      echo "Health file removed - app is now 'unhealthy'"
      # Keep running so we can see the restart
      sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
```

```bash
kubectl apply -f pod-liveness.yaml

# Watch the pod — it will restart after ~35 seconds
kubectl get pod liveness-demo -w

# Check restart count
kubectl describe pod liveness-demo | grep -A 5 "Last State"
```

### Step 4.2: Readiness Probe — Traffic Control

Create `pod-readiness.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-demo
  labels:
    app: readiness-demo
spec:
  containers:
  - name: app
    image: nginx:1.25
    ports:
    - containerPort: 80
    readinessProbe:
      httpGet:
        path: /ready
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 3
    # Create a custom endpoint for readiness
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "sleep 15 && echo 'ready' > /usr/share/nginx/html/ready"]
```

```bash
kubectl apply -f pod-readiness.yaml

# Watch — pod will be Running but NOT Ready for 15 seconds
kubectl get pod readiness-demo -w

# Create a service
kubectl expose pod readiness-demo --port=80 --name=readiness-svc

# Check endpoints — empty until ready
kubectl get endpoints readiness-svc -w
```

### Step 4.3: HTTP Probe with Real Backend

Create `pod-http-probe.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: http-probe-demo
spec:
  containers:
  - name: server
    image: python:3.11-slim
    command: ["python", "-c"]
    args:
    - |
      from http.server import HTTPServer, BaseHTTPRequestHandler
      import time

      start_time = time.time()

      class Handler(BaseHTTPRequestHandler):
          def do_GET(self):
              uptime = int(time.time() - start_time)
              if self.path == '/health':
                  self.send_response(200)
                  self.end_headers()
                  self.wfile.write(f'{{"status":"healthy","uptime":{uptime}}}'.encode())
              elif self.path == '/ready':
                  if uptime > 10:
                      self.send_response(200)
                      self.end_headers()
                      self.wfile.write(b'{"status":"ready"}')
                  else:
                      self.send_response(503)
                      self.end_headers()
                      self.wfile.write(f'{{"status":"warming up","remaining":{10-uptime}}}'.encode())
              else:
                  self.send_response(200)
                  self.end_headers()
                  self.wfile.write(f'Hello! Uptime: {uptime}s'.encode())
          def log_message(self, format, *args):
              pass  # Suppress logs

      HTTPServer(('', 8080), Handler).serve_forever()
    ports:
    - containerPort: 8080
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 3
      periodSeconds: 3
```

```bash
kubectl apply -f pod-http-probe.yaml
kubectl get pod http-probe-demo -w

# After ready, test the endpoints
kubectl port-forward pod/http-probe-demo 8080:8080 &
curl http://localhost:8080/health
curl http://localhost:8080/ready
kill %1
```

> **Checkpoint 4:** Understand the difference between liveness (restart on failure) and readiness (remove from service on failure).

---

## Part 5: Debugging Practice (10 minutes)

### Step 5.1: Debug a Failing Pod

Create `pod-broken.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken-pod
spec:
  containers:
  - name: app
    image: nginx:1.25
    command: ["/bin/sh", "-c", "exit 1"]    # Intentionally fails
```

```bash
kubectl apply -f pod-broken.yaml

# Observe the CrashLoopBackOff
kubectl get pod broken-pod -w

# Debug steps:
kubectl describe pod broken-pod
kubectl logs broken-pod
kubectl logs broken-pod --previous
```

### Step 5.2: Debug an Image Pull Error

```bash
kubectl run bad-image --image=nginx:nonexistent-tag

# Watch for ImagePullBackOff
kubectl get pod bad-image -w
kubectl describe pod bad-image | grep -A 10 Events
```

### Step 5.3: Debug with Ephemeral Container

```bash
# Create a running pod with minimal image
kubectl run minimal-app --image=nginx:1.25

# Attach a debug container (K8s 1.25+)
kubectl debug -it minimal-app --image=busybox:1.36 --target=minimal-app -- /bin/sh

# Inside debug container you can:
# - Check network: wget -qO- localhost:80
# - Check processes: ps aux
# - Check filesystem: ls /
```

---

## Cleanup

```bash
kubectl delete pod web-server resource-demo sidecar-demo ambassador-demo \
  adapter-demo init-demo app-with-migration liveness-demo readiness-demo \
  http-probe-demo broken-pod bad-image minimal-app
kubectl delete service readiness-svc
```

---

## Challenge Exercises

### Challenge 1: DockerComposeDemo Backend as a Pod
Create a pod manifest that runs the Node.js backend from this project:
1. Build the image and load it into minikube
2. Set environment variables for database connection
3. Add a liveness probe checking `/health`
4. Add a readiness probe
5. Set appropriate resource limits

### Challenge 2: Multi-Container Debugging
Create a pod with:
- Main container: nginx serving on port 80
- Sidecar: busybox that periodically curls the main container and logs response times
- Volume shared between them for results

### Challenge 3: Init Container Chain
Create a pod with 3 init containers that each write to a shared volume:
1. Init 1: Writes "Step 1 complete"
2. Init 2: Reads step 1 output, writes "Step 2 complete"  
3. Init 3: Reads steps 1-2, writes "All steps complete"
4. Main container: Reads and displays the final output

---

## Lab Review Checklist

- [ ] Created and accessed a basic pod
- [ ] Built a sidecar multi-container pod with shared volumes
- [ ] Implemented init containers for startup dependencies
- [ ] Configured liveness and readiness probes
- [ ] Successfully debugged a CrashLoopBackOff
- [ ] Successfully debugged an ImagePullBackOff
- [ ] Used kubectl exec, logs, describe for debugging

## Next Lab
→ [Lab 04: Deployments & ReplicaSets](../Module-04-Deployments/lab-04-deployments.md)
