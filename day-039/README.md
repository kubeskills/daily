## Day 39: Resource Contention - QoS Eviction Wars

## THE IDEA:
Deploy pods with different QoS classes (BestEffort, Burstable, Guaranteed) and trigger
memory pressure. Watch kubelet evict them in priority order.

## THE SETUP:
Fill a node with pods, consume all memory, and observe kubelet's eviction logic.
BestEffort dies first, then Burstable exceeding requests, finally Guaranteed.

## WHAT I LEARNED:
- QoS classes: Guaranteed (requests=limits), Burstable (has requests), BestEffort (none)
- Eviction order under pressure: BestEffort → Burstable (exceeding) → Guaranteed
- Pod priority can override QoS eviction order
- OOMKilled is different from eviction (container vs pod level)

## WHY IT MATTERS:
Resource contention causes:
- Critical pods evicted during load spikes
- Unpredictable performance degradation
- Cascading failures from eviction storms

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-039

Tomorrow: Certificate expiration and renewal failures.

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/playgrounds/scenario/kubernetes


---

# Killercoda Lab Instructions


## Step 1: Check node resources
```bash
kubectl describe node | grep -A 10 "Allocatable:"
kubectl describe node | grep -A 10 "Allocated resources:"
```

## Step 2: Deploy BestEffort pod (no requests/limits)
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
  labels:
    qos: besteffort
spec:
  containers:
  - name: app
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "100M"]
EOF
```

**Check QoS:**
```bash
kubectl get pod besteffort-pod -o jsonpath='{.status.qosClass}'
echo ""
```

Output: BestEffort

## Step 3: Deploy Burstable pod (requests < limits)
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
  labels:
    qos: burstable
spec:
  containers:
  - name: app
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "100M"]
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "500m"
EOF
```

**Check QoS:**
```bash
kubectl get pod burstable-pod -o jsonpath='{.status.qosClass}'
echo ""
```

Output: Burstable

## Step 4: Deploy Guaranteed pod (requests = limits)
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
  labels:
    qos: guaranteed
spec:
  containers:
  - name: app
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "100M"]
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "100m"
EOF
```

**Check QoS:**
```bash
kubectl get pod guaranteed-pod -o jsonpath='{.status.qosClass}'
echo ""
```

Output: Guaranteed

## Step 5: Monitor all QoS classes
```bash
kubectl get pods -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass,STATUS:.status.phase
```

## Step 6: Simulate memory pressure
```bash
# Deploy memory-hungry pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: memory-hog
spec:
  containers:
  - name: hog
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "800M"]
    resources:
      requests:
        memory: "500Mi"
      limits:
        memory: "1Gi"
EOF
```

**Watch for evictions:**
```bash
kubectl get pods -w
kubectl get events --sort-by='.lastTimestamp' | grep -i evict
```

## Step 7: Check node conditions
```bash
kubectl describe node | grep -A 5 "Conditions:"
```

Look for MemoryPressure: True/False

## Step 8: Test with PodPriority
```bash
cat <<EOF | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
globalDefault: false
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: false
EOF
```

**Deploy pods with priorities:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: low-priority-pod
spec:
  priorityClassName: low-priority
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "64Mi"
      limits:
        memory: "128Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: high-priority-pod
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "64Mi"
      limits:
        memory: "128Mi"
EOF
```

## Step 9: Check priority values
```bash
kubectl get pods -o custom-columns=NAME:.metadata.name,PRIORITY:.spec.priority,QOS:.status.qosClass
```

## Step 10: Test resource quota exhaustion
```bash
kubectl create namespace resource-test

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-quota
  namespace: resource-test
spec:
  hard:
    requests.memory: "1Gi"
    limits.memory: "2Gi"
EOF
```

**Try to exceed:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: exceed-quota
  namespace: resource-test
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "2Gi"
      limits:
        memory: "2Gi"
EOF
```

**Error:** Exceeds quota!

## Step 11: Test LimitRange defaults
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limits
  namespace: resource-test
spec:
  limits:
  - max:
      memory: "512Mi"
    min:
      memory: "64Mi"
    default:
      memory: "256Mi"
    defaultRequest:
      memory: "128Mi"
    type: Container
EOF
```

**Deploy without specifying resources:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: default-resources
  namespace: resource-test
spec:
  containers:
  - name: app
    image: nginx
EOF
```

**Check assigned resources:**
```bash
kubectl get pod default-resources -n resource-test -o jsonpath='{.spec.containers[0].resources}'
echo ""
```

Defaults applied!

## Step 12: Test CPU throttling (Burstable)
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cpu-throttled
spec:
  containers:
  - name: cpu-intensive
    image: polinux/stress
    command: ["stress"]
    args: ["--cpu", "2"]
    resources:
      requests:
        cpu: "100m"
      limits:
        cpu: "200m"
EOF
```

**Monitor CPU:**
```bash
kubectl top pod cpu-throttled
```

CPU capped at 200m despite trying to use 2 cores!

## Step 13: Test memory vs swap
```bash
kubectl describe node | grep -i swap
```

Swap should be disabled in Kubernetes!

## Step 14: Test pod eviction with finalizers
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: finalizer-pod
  finalizers:
  - example.com/block-eviction
spec:
  containers:
  - name: app
    image: nginx
EOF
```

**Try to delete:**
```bash
kubectl delete pod finalizer-pod &

# Check status
kubectl get pod finalizer-pod
```

Stuck in Terminating (finalizer blocks deletion)

**Remove finalizer:**
```bash
kubectl patch pod finalizer-pod -p '{"metadata":{"finalizers":[]}}'
```

## Step 15: Check kubelet eviction thresholds
```bash
# View kubelet config (if accessible)
# kubelet --help | grep -A 10 eviction

# Check node allocatable vs capacity
kubectl get node -o json | jq '.items[0].status | {capacity, allocatable}'
```

## Key Observations

✅ **QoS classes** - determine eviction priority
✅ **Eviction order** - BestEffort → Burstable (exceeding) → Guaranteed
✅ **Pod priority** - can override QoS order
✅ **OOMKilled** - container-level, happens before eviction
✅ **Resource requests** - guarantee minimum resources
✅ **Resource limits** - cap maximum usage

## Production Patterns

**Critical service (Guaranteed QoS + High Priority):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
spec:
  priorityClassName: system-cluster-critical
  containers:
  - name: app
    image: critical-app:v1
    resources:
      requests:
        memory: "1Gi"
        cpu: "500m"
      limits:
        memory: "1Gi"
        cpu: "500m"
```

**Background job (BestEffort + Low Priority):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: batch-job
spec:
  priorityClassName: low-priority
  containers:
  - name: worker
    image: batch-processor:v1
    # No resources = BestEffort
```

**Web service (Burstable):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-service
spec:
  containers:
  - name: web
    image: nginx
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
      limits:
        memory: "512Mi"
        cpu: "500m"
```

**Prevent eviction with PDB:**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: critical
```

## Cleanup
```bash
kubectl delete pod besteffort-pod burstable-pod guaranteed-pod memory-hog low-priority-pod high-priority-pod cpu-throttled finalizer-pod 2>/dev/null
kubectl delete priorityclass high-priority low-priority 2>/dev/null
kubectl delete namespace resource-test
```

---
