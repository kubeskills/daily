## Day 26: Kubelet Eviction - Resource Pressure

## THE IDEA:
Trigger node memory pressure and watch kubelet start evicting pods. BestEffort
pods die first, then Burstable, finally Guaranteed as a last resort.

## THE SETUP:
Deploy pods with different QoS classes, consume node resources, and observe
kubelet's eviction decisions based on priority and resource usage.

## WHAT I LEARNED:
- Kubelet evicts pods to prevent node failure
- Eviction order: BestEffort → Burstable (exceeding requests) → Guaranteed
- Soft eviction has grace period, hard eviction is immediate
- PodPriority affects eviction order

## WHY IT MATTERS:
Eviction issues cause:
- Unexpected pod terminations during load spikes
- Critical services evicted before batch jobs
- Cascading failures from resource pressure

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-026

Tomorrow: API server rate limiting blocking requests (Week 4 finale!).

---

# Killercoda Lab Instructions

## Step 1: Check node resource capacity

```bash
kubectl describe node | grep -A 10 "Allocatable:"
kubectl describe node | grep -A 10 "Allocated resources:"

```

## Step 2: Check kubelet eviction thresholds

```bash
# View kubelet config
kubectl get configmap kubelet-config -n kube-system -o yaml 2>/dev/null || \\
echo "Kubelet config not in ConfigMap"

# Default thresholds (if not customized):
# memory.available < 100Mi
# nodefs.available < 10%
# imagefs.available < 15%

```

## Step 3: Deploy pods with different QoS classes

```bash
cat <<EOF | kubectl apply -f -
# BestEffort - no requests or limits
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
    args: ["--vm", "1", "--vm-bytes", "50M", "--vm-hang", "1"]
---
# Burstable - requests < limits
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
    args: ["--vm", "1", "--vm-bytes", "50M", "--vm-hang", "1"]
    resources:
      requests:
        memory: "50Mi"
        cpu: "100m"
      limits:
        memory: "200Mi"
        cpu: "500m"
---
# Guaranteed - requests == limits
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
    args: ["--vm", "1", "--vm-bytes", "50M", "--vm-hang", "1"]
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
      limits:
        memory: "100Mi"
        cpu: "100m"
EOF

```

**Verify QoS classes:**

```bash
kubectl get pods -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass

```

## Step 4: Check current node memory usage

```bash
kubectl top node
kubectl describe node | grep -A 5 "Allocated resources:"

```

## Step 5: Trigger memory pressure (carefully!)

```bash
# Deploy memory-hungry pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: memory-eater
spec:
  containers:
  - name: stress
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "500M", "--vm-hang", "10"]
    resources:
      requests:
        memory: "100Mi"
      limits:
        memory: "600Mi"
EOF

```

**Watch for evictions:**

```bash
kubectl get pods -w
kubectl get events --sort-by='.lastTimestamp' | grep -i evict

```

## Step 6: Check eviction events

```bash
kubectl get events --sort-by='.lastTimestamp' | tail -20

```

Look for:

- "Evicted" reason
- "The node was low on resource: memory"

## Step 7: See which pod was evicted first

```bash
kubectl get pods -o wide
kubectl describe pod <evicted-pod-name> 2>/dev/null || echo "Pod already gone"

```

BestEffort pods evicted first!

## Step 8: Test with pod priority

```bash
# Create priority classes
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
  name: high-priority-pod
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "50Mi"
      limits:
        memory: "100Mi"
---
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
        memory: "50Mi"
      limits:
        memory: "100Mi"
EOF

```

## Step 9: Check pod priority values

```bash
kubectl get pods -o custom-columns=NAME:.metadata.name,PRIORITY:.spec.priority

```

## Step 10: Simulate disk pressure

```bash
# Create pod that fills disk
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: disk-filler
spec:
  containers:
  - name: filler
    image: busybox
    command:
    - sh
    - -c
    - |
      dd if=/dev/zero of=/tmp/bigfile bs=1M count=1000
      sleep 3600
    volumeMounts:
    - name: temp
      mountPath: /tmp
  volumes:
  - name: temp
    emptyDir: {}
EOF

```

**Monitor disk usage:**

```bash
kubectl exec disk-filler -- df -h /tmp

```

## Step 11: Check node conditions

```bash
kubectl describe node | grep -A 10 "Conditions:"

```

Look for:

- MemoryPressure: True/False
- DiskPressure: True/False
- PIDPressure: True/False

## Step 12: Test grace period eviction

```bash
# Soft eviction (with grace period)
# Hard eviction (immediate, no grace period)

# Check events for eviction type
kubectl get events --sort-by='.lastTimestamp' | grep -i evict

```

## Step 13: Monitor kubelet logs for eviction

```bash
# On node (if accessible):
# journalctl -u kubelet | grep -i evict

# Check via pod describe
kubectl get pods -A | grep Evicted
kubectl describe pod <evicted-pod> | grep -A 5 "Reason:"

```

## Step 14: Test PodDisruptionBudget with eviction

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: protected-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: protected
  template:
    metadata:
      labels:
        app: protected
    spec:
      containers:
      - name: app
        image: nginx
        resources:
          requests:
            memory: "50Mi"
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: protected-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: protected
EOF

```

**Note:** PDB does NOT protect against kubelet eviction (involuntary disruption)

## Step 15: Recovery after eviction

```bash
# Check pod restart count
kubectl get pods -o custom-columns=NAME:.metadata.name,RESTARTS:.status.containerStatuses[0].restartCount

# Deployment auto-recreates evicted pods
kubectl get deployment protected-app
kubectl get pods -l app=protected

```

## Key Observations

✅ **QoS eviction order** - BestEffort → Burstable → Guaranteed
✅ **Priority matters** - higher priority pods evicted last
✅ **PDB doesn't help** - eviction is involuntary (PDB only for voluntary)
✅ **Node conditions** - MemoryPressure/DiskPressure signal issues
✅ **Deployment recovery** - automatically recreates evicted pods

## Production Patterns

**Set proper resource requests:**

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"

```

**Use Guaranteed QoS for critical services:**

```yaml
resources:
  requests:
    memory: "1Gi"
    cpu: "500m"
  limits:
    memory: "1Gi"  # Same as requests
    cpu: "500m"

```

**Set pod priority for critical workloads:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
spec:
  priorityClassName: system-cluster-critical

```

**Monitor node conditions:**

```bash
# Alert on node pressure
kubectl get nodes -o json | jq '.items[] | select(.status.conditions[] | select(.type=="MemoryPressure" and .status=="True")) | .metadata.name'

```

## Cleanup

```bash
kubectl delete pod besteffort-pod burstable-pod guaranteed-pod memory-eater high-priority-pod low-priority-pod disk-filler 2>/dev/null
kubectl delete deployment protected-app 2>/dev/null
kubectl delete pdb protected-pdb 2>/dev/null
kubectl delete priorityclass high-priority low-priority 2>/dev/null

```

---

Next: Day 27 - API Server Rate Limiting