# Day 20: Node NotReady - The Eviction Cascade

## THE IDEA:
Simulate a node going NotReady and watch the pod eviction timer tick down. 
Explore tolerations, pod-eviction-timeout, and what happens to StatefulSets.

## THE SETUP:
Cordon a node, stop kubelet, and observe how long Kubernetes waits before 
evicting pods. Test different tolerations and pod types.

## WHAT I LEARNED:
- Default toleration: node.kubernetes.io/not-ready for 300s
- Pod eviction starts after tolerationSeconds expires
- StatefulSets pods get stuck in Terminating (wait for node recovery)
- DaemonSets tolerate NotReady indefinitely

## WHY IT MATTERS:
Node failure handling affects:
- Application downtime duration
- Data safety (StatefulSets)
- Cascading failures during partial outages

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-020

Tomorrow: Finalizers preventing resource deletion (Week 3 finale!).

---

## Killercoda Lab Instructions

### Step 1: Check initial node status

```bash
kubectl get nodes
kubectl describe node $(kubectl get nodes -o jsonpath='{.items[0].metadata.name}') | grep -A 5 Taints

```

Nodes should be Ready with no taints.

### Step 2: Deploy test pods

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: default-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: default-app
  template:
    metadata:
      labels:
        app: default-app
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: 100m
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: stateful-app
spec:
  serviceName: stateful
  replicas: 2
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
---
apiVersion: v1
kind: Service
metadata:
  name: stateful
spec:
  clusterIP: None
  selector:
    app: stateful
  ports:
  - port: 80
EOF

```

**Check pod placement:**

```bash
kubectl get pods -o wide

```

### Step 3: Check default tolerations

```bash
kubectl get pod $(kubectl get pods -l app=default-app -o jsonpath='{.items[0].metadata.name}') -o yaml | grep -A 10 tolerations

```

You'll see automatic tolerations for:

- `node.kubernetes.io/not-ready:NoExecute` for 300s
- `node.kubernetes.io/unreachable:NoExecute` for 300s

### Step 4: Simulate node failure (single-node cluster method)

```bash
# In multi-node cluster, cordon and simulate kubelet failure
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')

# Method 1: Stop kubelet (if you have node access)
# This won't work in Killercoda, so we'll simulate with taints

# Add NotReady taint manually to simulate
kubectl taint nodes $NODE node.kubernetes.io/not-ready:NoExecute --overwrite=true

```

**Watch pod status:**

```bash
kubectl get pods -w

```

Pods immediately show as Terminating, then get evicted!

### Step 5: Check eviction timing

```bash
kubectl describe pod $(kubectl get pods -l app=default-app -o jsonpath='{.items[0].metadata.name}') | grep -A 5 Events

```

Look for "[node.kubernetes.io/not-ready](http://node.kubernetes.io/not-ready)" toleration exceeded.

### Step 6: Remove taint and watch recovery

```bash
kubectl taint nodes $NODE node.kubernetes.io/not-ready:NoExecute-

```

**Watch pods reschedule:**

```bash
kubectl get pods -o wide -w

```

Deployment pods recreate on available nodes. StatefulSet pods recreate on original node.

### Step 7: Test custom toleration (longer wait)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: patient-pod
spec:
  tolerations:
  - key: node.kubernetes.io/not-ready
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 600  # Wait 10 minutes
  - key: node.kubernetes.io/unreachable
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 600
  containers:
  - name: nginx
    image: nginx
EOF

```

**Taint node again:**

```bash
kubectl taint nodes $NODE node.kubernetes.io/not-ready:NoExecute

```

**Watch different behavior:**

```bash
kubectl get pods -w

```

Deployment pods evict quickly, patient-pod stays for 10 minutes!

### Step 8: Test infinite toleration

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: immortal-pod
spec:
  tolerations:
  - key: node.kubernetes.io/not-ready
    operator: Exists
    effect: NoExecute
    # No tolerationSeconds = wait forever!
  - key: node.kubernetes.io/unreachable
    operator: Exists
    effect: NoExecute
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    kubernetes.io/hostname: $NODE
EOF

```

This pod NEVER evicts on NotReady!

### Step 9: Check StatefulSet behavior

```bash
# With node tainted, check StatefulSet
kubectl get statefulset stateful-app
kubectl get pods -l app=stateful

```

StatefulSet pods show Terminating but DON'T reschedule!
They're waiting for node to recover.

**Force delete if needed:**

```bash
kubectl delete pod stateful-app-0 --grace-period=0 --force

```

StatefulSet creates replacement immediately.

### Step 10: DaemonSet default behavior

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
        image: nginx
EOF

```

**Check DaemonSet tolerations:**

```bash
kubectl get pod $(kubectl get pods -l app=monitor -o jsonpath='{.items[0].metadata.name}') -o yaml | grep -A 20 tolerations

```

DaemonSets automatically tolerate NotReady indefinitely!

### Step 11: Test pod disruption during node drain

```bash
# Remove NotReady taint first
kubectl taint nodes $NODE node.kubernetes.io/not-ready:NoExecute- 2>/dev/null

# Wait for pods to stabilize
sleep 30

# Drain node (voluntary disruption)
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data --timeout=60s --grace-period=30

```

**Observe:**

- Deployment pods evict and reschedule elsewhere
- StatefulSet pods evict and reschedule on other nodes
- DaemonSet pods stay (due to --ignore-daemonsets)

### Step 12: Check pod deletion timestamp

```bash
kubectl get pods -o json | jq -r '.items[] | select(.metadata.deletionTimestamp != null) | {name: .metadata.name, deletionTimestamp: .metadata.deletionTimestamp, deletionGracePeriodSeconds: .metadata.deletionGracePeriodSeconds}'

```

Shows pods in Terminating state with deletion timestamp.

### Step 13: Uncordon and cleanup

```bash
kubectl uncordon $NODE
kubectl delete pod patient-pod immortal-pod 2>/dev/null
kubectl delete deployment default-app
kubectl delete statefulset stateful-app
kubectl delete daemonset node-monitor
kubectl delete service stateful

```

### Key Observations

✅ **Default tolerations** - 300s for NotReady/Unreachable
✅ **StatefulSets wait** - pods stuck in Terminating, waiting for node
✅ **DaemonSets persist** - tolerate NotReady indefinitely
✅ **Custom tolerations** - override default eviction timing
✅ **Force delete** - sometimes needed for stuck pods

### Production Patterns

**Critical system pods (longer wait):**

```yaml
tolerations:
- key: node.kubernetes.io/not-ready
  operator: Exists
  effect: NoExecute
  tolerationSeconds: 600  # 10 minutes
- key: node.kubernetes.io/unreachable
  operator: Exists
  effect: NoExecute
  tolerationSeconds: 600

```

**Fast failover for stateless apps:**

```yaml
tolerations:
- key: node.kubernetes.io/not-ready
  operator: Exists
  effect: NoExecute
  tolerationSeconds: 30  # Fail over quickly
- key: node.kubernetes.io/unreachable
  operator: Exists
  effect: NoExecute
  tolerationSeconds: 30

```

**StatefulSet with faster recovery:**

```yaml
apiVersion: apps/v1
kind: StatefulSet
spec:
  template:
    spec:
      tolerations:
      - key: node.kubernetes.io/not-ready
        operator: Exists
        effect: NoExecute
        tolerationSeconds: 120  # 2 minutes instead of 5

```

### Cleanup

```bash
kubectl delete deployment default-app 2>/dev/null
kubectl delete statefulset stateful-app 2>/dev/null
kubectl delete daemonset node-monitor 2>/dev/null
kubectl delete service stateful 2>/dev/null
kubectl delete pod patient-pod immortal-pod 2>/dev/null

```

---

Next: Day 21 - Finalizers Preventing Deletion (Week 3 Finale!)