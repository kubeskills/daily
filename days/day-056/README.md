## Day 56: Resource Exhaustion - Cluster at Capacity

### Email Copy

**Subject:** Day 56: Resource Exhaustion - When the Cluster is Full

THE IDEA:
Fill the cluster to capacity and watch new pods get stuck in Pending. Discover
resource fragmentation, bin-packing failures, and the scheduler giving up.

THE SETUP:
Deploy pods that consume all CPU/memory, trigger "Insufficient memory" errors,
test priority-based preemption, and observe scheduler back-off.

WHAT I LEARNED:
- Scheduler stops trying after back-off limit
- Resource fragmentation prevents large pod scheduling
- Priority preemption evicts lower-priority pods
- DaemonSets ignore capacity (scheduled anyway)
- Overcommitment allows more pods than capacity

WHY IT MATTERS:
Resource exhaustion causes:
- New workloads stuck in Pending indefinitely
- Autoscaling useless (no capacity to scale into)
- Critical services unable to restart

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-056

Tomorrow: Pod security standards enforcement failures.

---
Chad




---

## Killercoda Lab Instructions

## Step 1: Check cluster capacity

```bash
# Get node allocatable resources
kubectl describe nodes | grep -A 5 "Allocatable:"

# Get current allocation
kubectl describe nodes | grep -A 10 "Allocated resources:"

# Summary view
kubectl top nodes
```

## Step 2: Deploy pods to fill capacity

```bash
# Create namespace
kubectl create namespace capacity-test

# Deploy many pods requesting resources
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-hog
  namespace: capacity-test
spec:
  replicas: 20
  selector:
    matchLabels:
      app: hog
  template:
    metadata:
      labels:
        app: hog
    spec:
      containers:
      - name: app
        image: nginx
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
EOF
```

**Watch scheduling:**
```bash
kubectl get pods -n capacity-test -w
```

## Step 3: Check for Pending pods

```bash
# Find Pending pods
kubectl get pods -n capacity-test --field-selector=status.phase=Pending

# Check why Pending
PENDING_POD=$(kubectl get pods -n capacity-test --field-selector=status.phase=Pending -o jsonpath='{.items[0].metadata.name}')

if [ -n "$PENDING_POD" ]; then
  kubectl describe pod -n capacity-test $PENDING_POD | grep -A 10 "Events:"
fi
```

Shows "Insufficient cpu" or "Insufficient memory"!

## Step 4: Test resource fragmentation

```bash
# Delete deployment
kubectl delete deployment resource-hog -n capacity-test

# Create pods with varying sizes
for size in 100m 200m 500m 1000m; do
  kubectl run frag-$size -n capacity-test \
    --image=nginx \
    --requests="cpu=$size,memory=256Mi" \
    --limits="cpu=$size,memory=512Mi"
done

sleep 10

# Try to schedule large pod
kubectl run large-pod -n capacity-test \
  --image=nginx \
  --requests="cpu=2000m,memory=2Gi" \
  --limits="cpu=2000m,memory=2Gi"

# Check status
kubectl get pod large-pod -n capacity-test
kubectl describe pod large-pod -n capacity-test | grep -A 5 "Events:"
```

Fragmentation prevents large pod scheduling!

## Step 5: Test priority-based preemption

```bash
# Create high-priority class
cat <<EOF | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
globalDefault: false
description: "High priority for critical workloads"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: false
description: "Low priority for batch workloads"
EOF

# Deploy low-priority pods
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: low-priority-app
  namespace: capacity-test
spec:
  replicas: 10
  selector:
    matchLabels:
      app: low-pri
  template:
    metadata:
      labels:
        app: low-pri
    spec:
      priorityClassName: low-priority
      containers:
      - name: app
        image: nginx
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
EOF

kubectl wait --for=condition=Ready pods -l app=low-pri -n capacity-test --timeout=120s

# Deploy high-priority pod (should preempt low-priority)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: high-priority-pod
  namespace: capacity-test
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "1000m"
        memory: "1Gi"
EOF

# Watch preemption
kubectl get pods -n capacity-test -w
```

Low-priority pods evicted to make room!

## Step 6: Test scheduler back-off

```bash
# Create impossible-to-schedule pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: impossible-pod
  namespace: capacity-test
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "100000m"  # 100 CPUs!
        memory: "1Ti"
EOF

# Check events
sleep 30
kubectl describe pod impossible-pod -n capacity-test | grep -A 20 "Events:"
```

Scheduler gives up after retries!

## Step 7: Test DaemonSet ignores capacity

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: resource-daemon
  namespace: capacity-test
spec:
  selector:
    matchLabels:
      app: daemon
  template:
    metadata:
      labels:
        app: daemon
    spec:
      containers:
      - name: app
        image: nginx
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
EOF

# Check DaemonSet pods scheduled despite capacity
kubectl get pods -n capacity-test -l app=daemon -o wide
```

DaemonSets bypass capacity limits!

## Step 8: Test overcommitment

```bash
# Check total requests vs allocatable
kubectl describe nodes | grep -E "Allocatable:|Allocated resources:" -A 5

# Pods can use limits beyond requests (overcommit)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: overcommit-pod
  namespace: capacity-test
spec:
  containers:
  - name: app
    image: polinux/stress
    command: ["stress"]
    args: ["--cpu", "4"]
    resources:
      requests:
        cpu: "100m"  # Small request
      limits:
        cpu: "4000m"  # Large limit (overcommit)
EOF
```

## Step 9: Test resource quotas

```bash
# Create ResourceQuota
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: capacity-quota
  namespace: capacity-test
spec:
  hard:
    requests.cpu: "5"
    requests.memory: "10Gi"
    limits.cpu: "10"
    limits.memory: "20Gi"
    pods: "20"
EOF

# Check quota usage
kubectl describe resourcequota capacity-quota -n capacity-test
```

## Step 10: Test LimitRange defaults

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: capacity-limits
  namespace: capacity-test
spec:
  limits:
  - max:
      cpu: "2"
      memory: "4Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    type: Container
EOF

# Deploy pod without resources specified
kubectl run auto-sized -n capacity-test --image=nginx

# Check assigned resources
kubectl get pod auto-sized -n capacity-test -o jsonpath='{.spec.containers[0].resources}'
echo ""
```

## Step 11: Check node taints and tolerations

```bash
# Check node taints
kubectl describe nodes | grep "Taints:"

# Pods without matching tolerations can't schedule on tainted nodes
```

## Step 12: Test affinity constraints

```bash
# Create pod with impossible affinity
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: affinity-fail
  namespace: capacity-test
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: nonexistent-label
            operator: In
            values:
            - "true"
  containers:
  - name: app
    image: nginx
EOF

# Check why not scheduled
kubectl describe pod affinity-fail -n capacity-test | grep -A 10 "Events:"
```

No nodes match affinity!

## Step 13: Monitor scheduler metrics

```bash
# Check scheduler logs
kubectl logs -n kube-system -l component=kube-scheduler --tail=50 | grep -i "unschedulable\|insufficient"

# Check pending pod metrics
kubectl get pods -A --field-selector=status.phase=Pending | wc -l
```

## Step 14: Test cluster autoscaler triggers

```bash
# In cloud environments, pending pods trigger autoscaling
echo "Cluster Autoscaler triggers:"
echo "- Pending pods due to insufficient resources"
echo "- Respects PodDisruptionBudgets"
echo "- Won't scale up for DaemonSet pods"
echo "- Scales down underutilized nodes"
```

## Step 15: Capacity planning report

```bash
cat > /tmp/capacity-report.sh << 'EOF'
#!/bin/bash
echo "=== Cluster Capacity Report ==="

echo -e "\n1. Node Resources:"
kubectl top nodes

echo -e "\n2. Total Allocatable:"
kubectl describe nodes | grep "Allocatable:" -A 5 | grep -E "cpu:|memory:"

echo -e "\n3. Total Allocated:"
kubectl describe nodes | grep "Allocated resources:" -A 10 | tail -5

echo -e "\n4. Pending Pods:"
kubectl get pods -A --field-selector=status.phase=Pending | tail -n +2 | wc -l

echo -e "\n5. Resource Requests by Namespace:"
kubectl get pods -A -o json | jq -r '
  .items |
  group_by(.metadata.namespace) |
  map({
    namespace: .[0].metadata.namespace,
    cpu_requests: (map(.spec.containers[].resources.requests.cpu // "0" | gsub("m";"") | tonumber) | add),
    memory_requests: (map(.spec.containers[].resources.requests.memory // "0Mi" | gsub("Mi|Gi";"") | tonumber) | add)
  }) |
  .[] |
  "\(.namespace): \(.cpu_requests)m CPU, \(.memory_requests)Mi RAM"
'

echo -e "\n6. Largest Pods:"
kubectl get pods -A -o json | jq -r '
  .items |
  map({
    name: "\(.metadata.namespace)/\(.metadata.name)",
    cpu: (.spec.containers[].resources.requests.cpu // "0m"),
    memory: (.spec.containers[].resources.requests.memory // "0Mi")
  }) |
  sort_by(.cpu) |
  reverse |
  .[:5] |
  .[] |
  "\(.name): \(.cpu) CPU, \(.memory) RAM"
'
EOF

chmod +x /tmp/capacity-report.sh
/tmp/capacity-report.sh
```

## Key Observations

✅ **Pending pods** - scheduler can't find suitable nodes
✅ **Fragmentation** - small gaps prevent large pod placement
✅ **Priority preemption** - high-priority evicts low-priority
✅ **DaemonSets** - ignore capacity constraints
✅ **Overcommitment** - limits exceed requests (CPU throttling)
✅ **Scheduler back-off** - stops retrying impossible pods

## Production Patterns

**Resource requests and limits:**
```yaml
# Right-sized application
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          requests:
            cpu: "250m"     # Guaranteed minimum
            memory: "512Mi"
          limits:
            cpu: "1000m"    # Burst capacity
            memory: "1Gi"   # Hard limit
```

**Priority classes for critical workloads:**
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-critical
value: 10000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Production critical services"
---
apiVersion: v1
kind: Pod
spec:
  priorityClassName: production-critical
```

**Resource quotas per namespace:**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    persistentvolumeclaims: "10"
    pods: "50"
```

**Monitoring alerts:**
```yaml
# Prometheus alerts
- alert: ClusterNearCapacity
  expr: |
    sum(kube_pod_container_resource_requests{resource="cpu"}) /
    sum(kube_node_status_allocatable{resource="cpu"}) > 0.85
  for: 10m
  annotations:
    summary: "Cluster CPU capacity at 85%"

- alert: PodsPendingTooLong
  expr: |
    sum(kube_pod_status_phase{phase="Pending"}) > 5
  for: 15m
  annotations:
    summary: "More than 5 pods pending for 15+ minutes"
```

## Cleanup

```bash
kubectl delete namespace capacity-test
kubectl delete priorityclass high-priority low-priority
rm -f /tmp/capacity-report.sh
```

---
Next: Day 57 - Pod Security Standards Enforcement Failures
