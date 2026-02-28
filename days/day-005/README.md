# Day 5: Taints and Tolerations - Dedicated Node Failure

## THE IDEA:
Taint a node with NoExecute and watch pods get evicted immediately. Then create
pods with matching tolerations to survive the taint. Test what happens when
tolerations expire.

## THE SETUP:
Label a node, taint it with NoExecute, and observe running pods getting evicted.
Deploy new pods with tolerations and test time-based eviction.

## WHAT I LEARNED:
- NoExecute evicts immediately, NoSchedule prevents future scheduling
- Tolerations can have tolerationSeconds for time-based eviction
- Node taints survive node restarts (stored in etcd)
- DaemonSets automatically tolerate most taints

## WHY IT MATTERS:
Taints enable:
- Dedicated nodes for specific workloads (GPU, high-memory)
- Node draining without manual pod deletion
- Isolating problematic nodes without removing them

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-005

---

## Killercoda Lab Instructions

Step 1: Deploy pods on all nodes

```
# Create multiple pods
kubectl create deployment nginx --image=nginx --replicas=3

```

**Check pod placement:**

```bash
kubectl get pods -o wide

```

Note which node each pod is on.

**Step 2: Taint a node with NoExecute**

Get node name:

```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
echo "Tainting node: $NODE"

```

Apply the taint:

```bash
kubectl taint nodes $NODE dedicated=gpu:NoExecute

```

**Watch pods evacuate:**

```bash
kubectl get pods -o wide -w

```

Pods on the tainted node get terminated and rescheduled elsewhere immediately!

**Step 3: Check the taint**

```bash
kubectl describe node $NODE | grep Taints

```

Output: `Taints: dedicated=gpu:NoExecute`

**Step 4: Try to schedule a pod without toleration**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-toleration
spec:
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    kubernetes.io/hostname: $NODE
EOF

```

**Check status:**

```bash
kubectl get pod no-toleration

```

Status: Pending (can’t schedule on tainted node)

```bash
kubectl describe pod no-toleration | grep -A 3 Events

```

You’ll see: “0/X nodes are available: 1 node(s) had untolerated taint”

**Step 5: Create pod with matching toleration**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoExecute"
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    kubernetes.io/hostname: $NODE
EOF

```

**Verify it schedules:**

```bash
kubectl get pod with-toleration -o wide

```

Success! Pod runs on the tainted node.

**Step 6: Test time-based eviction**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: temporary-toleration
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoExecute"
    tolerationSeconds: 30  # Evict after 30 seconds
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    kubernetes.io/hostname: $NODE
EOF

```

**Watch it get evicted after 30 seconds:**

```bash
kubectl get pod temporary-toleration -w

```

Pod starts Running, then after 30s gets Terminating.

**Step 7: Test NoSchedule vs NoExecute**

Remove the NoExecute taint, add NoSchedule:

```bash
kubectl taint nodes $NODE dedicated=gpu:NoExecute-  # Remove
kubectl taint nodes $NODE dedicated=gpu:NoSchedule  # Add

```

**Deploy a pod on that node manually:**

```bash
kubectl run test-pod --image=nginx --overrides='
{
  "spec": {
    "nodeSelector": {
      "kubernetes.io/hostname": "'$NODE'"
    }
  }
}'

```

**Check status:**

```bash
kubectl get pod test-pod

```

Stays Pending. NoSchedule prevents new pods but doesn’t evict existing ones.

**Step 8: View common system taints**

```bash
kubectl get nodes -o json | jq -r '.items[].spec.taints'

```

Common taints:

- `node.kubernetes.io/not-ready:NoExecute` (automatically added when node is NotReady)
- `node.kubernetes.io/unreachable:NoExecute` (node controller adds this)
- `node.kubernetes.io/disk-pressure:NoSchedule`
- `node.kubernetes.io/memory-pressure:NoSchedule`

**Key Observations**

✅ **NoExecute** - evicts running pods immediately
✅ **NoSchedule** - prevents new pods, keeps existing ones
✅ **PreferNoSchedule** - soft version, scheduler tries to avoid but not required
✅ **tolerationSeconds** - allows temporary toleration with delayed eviction

**Production Patterns**

**Dedicated GPU nodes:**

```yaml
tolerations:
- key: nvidia.com/gpu
  operator: Exists
  effect: NoSchedule

```

**Tolerate spot instances with 2-min grace:**

```yaml
tolerations:
- key: spot
  operator: Equal
  value: "true"
  effect: NoExecute
  tolerationSeconds: 120

```

**DaemonSet that runs everywhere (monitoring):**

```yaml
tolerations:
- operator: Exists  # Tolerates all taints

```

### Cleanup

```bash
kubectl taint nodes $NODE dedicated=gpu:NoSchedule-
kubectl delete pod no-toleration with-toleration temporary-toleration test-pod
kubectl delete deployment nginx

```

---

Next: Week 2 Preview - Liveness probe death loops, init container failures, and PVC binding chaos