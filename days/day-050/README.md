## Day 50: Node Failures - Eviction Storm Chaos

### Email Copy

**Subject:** Day 50: Node Failures - When Nodes Die and Pods Scatter

THE IDEA:
Simulate a node failure and watch Kubernetes struggle to reschedule pods. Discover
NotReady toleration delays, StatefulSet stuck pods, and cascading eviction storms.

THE SETUP:
Drain nodes, trigger NotReady state, observe pod eviction timing, and test what
happens when multiple nodes fail simultaneously.

WHAT I LEARNED:
- Pods tolerate NotReady for 300s before eviction (default)
- StatefulSet pods stuck in Terminating (wait for node recovery)
- DaemonSets tolerate node failures indefinitely
- Pod Disruption Budgets can block evictions
- Too many evictions overwhelm scheduler

WHY IT MATTERS:
Node failure handling issues cause:
- Extended downtime (waiting for eviction timeout)
- Split-brain in stateful applications
- Resource exhaustion from eviction storms

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-050

Tomorrow: API server overload and request throttling.

---
Chad


---

## Killercoda Lab Instructions


## Step 1: Check default node tolerations

```bash
# Deploy test pod
kubectl run node-test --image=nginx

# Check default tolerations
kubectl get pod node-test -o jsonpath='{.spec.tolerations}' | jq .
```

Shows default tolerations for node.kubernetes.io/not-ready and node.kubernetes.io/unreachable (300s)

## Step 2: Check node status

```bash
kubectl get nodes
kubectl describe nodes | grep -A 5 "Conditions:"
```

## Step 3: Deploy application across nodes

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-node-app
spec:
  replicas: 6
  selector:
    matchLabels:
      app: multi-node
  template:
    metadata:
      labels:
        app: multi-node
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
EOF
```

**Check pod distribution:**
```bash
kubectl get pods -l app=multi-node -o wide
```

## Step 4: Cordon node (prevent new scheduling)

```bash
# Get node name
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')

# Cordon node
kubectl cordon $NODE

# Check node status
kubectl get nodes
```

Shows SchedulingDisabled

## Step 5: Drain node gracefully

```bash
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data --grace-period=30

# Watch pods reschedule
kubectl get pods -l app=multi-node -o wide -w
```

## Step 6: Uncordon node

```bash
kubectl uncordon $NODE

# Pods don't automatically rebalance!
kubectl get pods -l app=multi-node -o wide
```

## Step 7: Simulate node NotReady

```bash
# Mark node as unschedulable and taint it
kubectl taint nodes $NODE node.kubernetes.io/unreachable:NoExecute --overwrite

# Check node condition
kubectl get nodes

# Watch pod eviction (takes ~300 seconds)
kubectl get pods -l app=multi-node -o wide -w
```

**Check toleration timeout:**
```bash
kubectl get pod -l app=multi-node -o jsonpath='{.items[0].spec.tolerations}' | jq '.[] | select(.key=="node.kubernetes.io/unreachable")'
```

## Step 8: Remove taint to recover

```bash
kubectl taint nodes $NODE node.kubernetes.io/unreachable:NoExecute-

# Node becomes Ready again
kubectl get nodes
```

## Step 9: Test StatefulSet stuck in Terminating

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: stateful-svc
spec:
  clusterIP: None
  selector:
    app: stateful
  ports:
  - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: stateful-app
spec:
  serviceName: stateful-svc
  replicas: 3
  selector:
    matchLabels:
      app: stateful
  template:
    metadata:
      labels:
        app: stateful
    spec:
      containers:
      - name: nginx
        image: nginx
EOF

kubectl wait --for=condition=Ready pods -l app=stateful --timeout=60s

# Taint node where stateful pod runs
STATEFUL_NODE=$(kubectl get pod stateful-app-0 -o jsonpath='{.spec.nodeName}')
kubectl taint nodes $STATEFUL_NODE node.kubernetes.io/unreachable:NoExecute --overwrite

# Watch pod get stuck
kubectl get pods -l app=stateful -w
```

StatefulSet pod stuck in Terminating!

## Step 10: Force delete stuck pod

```bash
# After waiting, force delete
kubectl delete pod stateful-app-0 --grace-period=0 --force

# Check if new pod starts
kubectl get pods -l app=stateful -w
```

## Step 11: Test DaemonSet tolerance

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitor
spec:
  selector:
    matchLabels:
      app: monitor
  template:
    metadata:
      labels:
        app: monitor
    spec:
      containers:
      - name: monitor
        image: busybox
        command: ['sh', '-c', 'while true; do echo "Monitoring..."; sleep 60; done']
EOF

# Check tolerations
kubectl get pod -l app=monitor -o jsonpath='{.items[0].spec.tolerations}' | jq .
```

DaemonSets tolerate NotReady indefinitely!

## Step 12: Test PodDisruptionBudget blocking eviction

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
      - name: nginx
        image: nginx
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: protected-pdb
spec:
  minAvailable: 3  # All 3 must be available!
  selector:
    matchLabels:
      app: protected
EOF

kubectl wait --for=condition=Ready pods -l app=protected --timeout=60s

# Try to drain node with protected pods
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data --timeout=30s 2>&1 || echo "Drain blocked by PDB!"
```

PDB blocks drain!

## Step 13: Test eviction with custom toleration

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: fast-eviction
spec:
  tolerations:
  - key: node.kubernetes.io/not-ready
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 10  # Evict after 10s!
  containers:
  - name: nginx
    image: nginx
EOF

# Taint node
kubectl taint nodes $NODE node.kubernetes.io/not-ready:NoExecute --overwrite

# Watch fast eviction
kubectl get pod fast-eviction -w
```

Evicted in 10 seconds!

## Step 14: Simulate multiple node failures

```bash
# Get all nodes
NODES=$(kubectl get nodes -o jsonpath='{.items[*].metadata.name}')

# Taint all nodes (careful!)
for node in $NODES; do
  kubectl taint nodes $node node.kubernetes.io/unreachable:NoExecute --overwrite &
done

# Watch eviction storm
kubectl get pods -A -w
```

**Cleanup taints:**
```bash
for node in $NODES; do
  kubectl taint nodes $node node.kubernetes.io/unreachable:NoExecute- 2>/dev/null
done
```

## Step 15: Check node pressure eviction

```bash
# Check node conditions
kubectl describe nodes | grep -E "MemoryPressure|DiskPressure|PIDPressure"

# Check kubelet eviction thresholds
kubectl get --raw /api/v1/nodes/$NODE/proxy/configz | jq '.kubeletconfig.evictionHard' 2>/dev/null || echo "Configz not available"
```

## Key Observations

✅ **NotReady tolerance** - 300s default before eviction
✅ **StatefulSet pods** - stuck in Terminating waiting for node
✅ **DaemonSets** - tolerate node failures indefinitely
✅ **PDB** - can block voluntary evictions (not involuntary)
✅ **Force delete** - required for stuck StatefulSet pods
✅ **Eviction storms** - overwhelm scheduler with many simultaneous failures

## Production Patterns

**Fast failover for stateless apps:**
```yaml
apiVersion: v1
kind: Pod
spec:
  tolerations:
  - key: node.kubernetes.io/not-ready
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 30  # Fast eviction
  - key: node.kubernetes.io/unreachable
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 30
```

**Slow eviction for stateful apps:**
```yaml
apiVersion: v1
kind: Pod
spec:
  tolerations:
  - key: node.kubernetes.io/not-ready
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 600  # Wait 10 minutes
```

**Protected critical workload:**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      tier: critical
```

**Safe node drain procedure:**
```bash
#!/bin/bash
NODE=$1

# Check if node has critical pods
CRITICAL=$(kubectl get pods --field-selector spec.nodeName=$NODE \
  -o json | jq -r '.items[] | select(.metadata.labels.tier=="critical") | .metadata.name')

if [ -n "$CRITICAL" ]; then
  echo "Critical pods found: $CRITICAL"
  echo "Proceed with caution!"
fi

# Cordon first
kubectl cordon $NODE

# Drain with safety checks
kubectl drain $NODE \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=300 \
  --timeout=600s \
  --pod-selector='tier!=critical'

echo "Node drained successfully"
```

**Monitor node health:**
```yaml
# Prometheus alerts
- alert: NodeNotReady
  expr: kube_node_status_condition{condition="Ready",status="true"} == 0
  for: 5m
  annotations:
    summary: "Node {{ $labels.node }} not ready"

- alert: TooManyPodsEvicted
  expr: rate(kube_pod_status_phase{phase="Failed"}[5m]) > 10
  annotations:
    summary: "Eviction storm detected"
```

## Cleanup

```bash
# Remove all taints
for node in $(kubectl get nodes -o name); do
  kubectl taint $node node.kubernetes.io/unreachable:NoExecute- 2>/dev/null
  kubectl taint $node node.kubernetes.io/not-ready:NoExecute- 2>/dev/null
  kubectl uncordon ${node#node/} 2>/dev/null
done

kubectl delete deployment multi-node-app protected-app
kubectl delete statefulset stateful-app
kubectl delete service stateful-svc
kubectl delete daemonset node-monitor
kubectl delete pdb protected-pdb
kubectl delete pod node-test fast-eviction 2>/dev/null
```

---
Next: Day 51 - API Server Overload and Request Throttling
