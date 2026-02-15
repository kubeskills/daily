## Day 44: Resource Metrics - Debugging Slow Pods

## THE IDEA:
Deploy pods with various performance issues and use kubectl top, metrics-server,
and cAdvisor to identify CPU throttling, memory pressure, and I/O bottlenecks.

## THE SETUP:
Create CPU-bound, memory-bound, and I/O-bound pods. Use metrics to diagnose
which resource is the bottleneck and why performance degrades.

## WHAT I LEARNED:
- kubectl top shows current usage (not historical)
- CPU throttling happens at container level
- Memory pressure triggers OOM before showing in metrics
- Disk I/O isn't visible in kubectl top
- Requests vs limits affect scheduling vs throttling

## WHY IT MATTERS:
Performance issues without metrics cause:
- Guessing which resource to scale
- Wasted money on wrong instance types
- Undetected resource starvation

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-044

Tomorrow: kubectl debug and ephemeral containers.


---

# Killercoda Lab Instructions


## Step 1: Verify metrics-server is installed

```bash
kubectl get deployment metrics-server -n kube-system

# If not installed:
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Patch for local development (allows insecure TLS)
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

# Wait for ready
kubectl wait --for=condition=Ready pods -l k8s-app=metrics-server -n kube-system --timeout=120s
```

## Step 2: Check basic metrics

```bash
kubectl top nodes
kubectl top pods -A
```

## Step 3: Deploy CPU-bound pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cpu-bound
spec:
  containers:
  - name: stress
    image: polinux/stress
    command: ["stress"]
    args: ["--cpu", "2", "--timeout", "600s"]
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "200m"  # Will throttle!
        memory: "128Mi"
EOF
```

**Monitor CPU usage:**
```bash
watch -n 2 "kubectl top pod cpu-bound"
```

CPU hits 200m limit and gets throttled!

## Step 4: Check CPU throttling

```bash
# Check container stats
kubectl exec cpu-bound -- cat /sys/fs/cgroup/cpu/cpu.stat 2>/dev/null || \
kubectl exec cpu-bound -- cat /sys/fs/cgroup/cpu.stat 2>/dev/null || \
echo "CGroup stats not accessible"

# Observe throttling in pod metrics
kubectl top pod cpu-bound --containers
```

## Step 5: Deploy memory-bound pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: memory-bound
spec:
  containers:
  - name: stress
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "500M", "--vm-hang", "1"]
    resources:
      requests:
        cpu: "100m"
        memory: "256Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
EOF
```

**Monitor memory:**
```bash
watch -n 2 "kubectl top pod memory-bound"
```

## Step 6: Trigger OOMKill

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oom-test
spec:
  containers:
  - name: stress
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "600M"]
    resources:
      requests:
        memory: "256Mi"
      limits:
        memory: "512Mi"  # Tries to allocate 600M!
EOF
```

**Check status:**
```bash
kubectl get pod oom-test -w
kubectl describe pod oom-test | grep -A 5 "Last State"
```

Shows OOMKilled!

## Step 7: Deploy I/O-bound pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: io-bound
spec:
  containers:
  - name: stress
    image: polinux/stress
    command: ["stress"]
    args: ["--io", "4", "--timeout", "600s"]
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "500m"
        memory: "128Mi"
EOF
```

**Monitor (I/O not shown in kubectl top!):**
```bash
kubectl top pod io-bound
# CPU usage is low, but pod is busy with I/O
```

## Step 8: Compare pods with different QoS classes

```bash
# BestEffort (no requests/limits)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: besteffort
spec:
  containers:
  - name: app
    image: nginx
EOF
```

```bash
# Burstable (requests < limits)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: burstable
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
EOF
```

```bash
# Guaranteed (requests = limits)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "200m"
        memory: "256Mi"
      limits:
        cpu: "200m"
        memory: "256Mi"
EOF
```

**Compare metrics:**
```bash
kubectl top pods besteffort burstable guaranteed
kubectl get pods besteffort burstable guaranteed -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass
```

## Step 9: Check historical metrics with Metrics API

```bash
# Get raw metrics from API
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes" | jq .
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/pods" | jq '.items[0]'
```

## Step 10: Test resource contention

```bash
# Deploy multiple CPU-hungry pods
for i in {1..5}; do
  kubectl run cpu-hog-$i --image=polinux/stress \
    --restart=Never \
    --requests='cpu=100m,memory=64Mi' \
    --limits='cpu=200m,memory=128Mi' \
    -- stress --cpu 2 --timeout 300s
done

# Monitor node resources
watch -n 2 "kubectl top nodes; echo '---'; kubectl top pods | grep cpu-hog"
```

## Step 11: Check pod resource utilization percentage

```bash
# Get pod with resources
POD_CPU_USAGE=$(kubectl top pod guaranteed --no-headers | awk '{print $2}')
POD_CPU_REQUEST=$(kubectl get pod guaranteed -o jsonpath='{.spec.containers[0].resources.requests.cpu}')
POD_CPU_LIMIT=$(kubectl get pod guaranteed -o jsonpath='{.spec.containers[0].resources.limits.cpu}')

echo "CPU Usage: $POD_CPU_USAGE"
echo "CPU Request: $POD_CPU_REQUEST"
echo "CPU Limit: $POD_CPU_LIMIT"
```

## Step 12: Test metrics for multi-container pods

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  containers:
  - name: app1
    image: polinux/stress
    command: ["stress"]
    args: ["--cpu", "1"]
    resources:
      requests:
        cpu: "50m"
      limits:
        cpu: "100m"
  - name: app2
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "100M"]
    resources:
      requests:
        memory: "64Mi"
      limits:
        memory: "128Mi"
EOF
```

**Check per-container metrics:**
```bash
kubectl top pod multi-container --containers
```

## Step 13: Check metrics-server refresh interval

```bash
# Metrics refresh every ~60 seconds
kubectl get deployment metrics-server -n kube-system -o yaml | grep -A 5 args
```

## Step 14: Test autoscaling based on metrics

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
EOF
```

**Check HPA status:**
```bash
kubectl get hpa web-hpa
kubectl describe hpa web-hpa
```

## Step 15: Debug performance with metrics

```bash
# Identify resource-hungry pods
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory

# Check node pressure
kubectl describe nodes | grep -A 5 "Allocated resources"

# Find pods hitting limits
kubectl get pods -A -o json | jq -r '
  .items[] |
  select(.spec.containers[].resources.limits.cpu != null) |
  "\(.metadata.namespace)/\(.metadata.name)"
'
```

## Key Observations

✅ **kubectl top** - shows current usage (not historical)
✅ **CPU throttling** - happens when usage exceeds limits
✅ **OOMKill** - instant when memory limit exceeded
✅ **I/O bottlenecks** - not visible in kubectl top
✅ **Metrics refresh** - ~60 second intervals
✅ **QoS classes** - affect scheduling and eviction

## Production Patterns

**Properly sized pod:**
```yaml
resources:
  requests:
    cpu: "100m"      # Guaranteed minimum
    memory: "256Mi"
  limits:
    cpu: "500m"      # Burst capacity
    memory: "512Mi"  # Hard limit (OOM if exceeded)
```

**CPU-intensive workload:**
```yaml
resources:
  requests:
    cpu: "1000m"  # 1 CPU guaranteed
    memory: "512Mi"
  limits:
    cpu: "2000m"  # Can burst to 2 CPUs
    memory: "1Gi"
```

**Memory-intensive workload:**
```yaml
resources:
  requests:
    cpu: "100m"
    memory: "2Gi"   # Guaranteed 2GB
  limits:
    cpu: "500m"
    memory: "2Gi"   # No burst (Guaranteed QoS)
```

**Monitoring alerting:**
```yaml
# Prometheus alerts
- alert: HighCPUThrottling
  expr: |
    rate(container_cpu_cfs_throttled_seconds_total[5m]) > 0.25
  for: 5m
  annotations:
    summary: "Pod {{ $labels.pod }} is being CPU throttled"

- alert: PodNearMemoryLimit
  expr: |
    container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.9
  for: 5m
  annotations:
    summary: "Pod {{ $labels.pod }} using 90% of memory limit"
```

## Cleanup

```bash
kubectl delete pod cpu-bound memory-bound oom-test io-bound besteffort burstable guaranteed multi-container
kubectl delete pod cpu-hog-{1..5} 2>/dev/null
kubectl delete deployment web-app
kubectl delete hpa web-hpa
```

---
