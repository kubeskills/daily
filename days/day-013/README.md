# Day 13: PodDisruptionBudgets - Blocking Maintenance

## THE IDEA:
Create a PodDisruptionBudget that requires 100% availability. Then try to drain
a node and watch kubectl hang forever, unable to evict pods.

## THE SETUP:
Deploy a 2-replica deployment with a PDB requiring minAvailable: 2. Attempt node
drain and discover your maintenance window is now infinite.

## WHAT I LEARNED:
- PDBs block voluntary disruptions (drain, eviction) not involuntary (node crash)
- minAvailable vs maxUnavailable: different ways to express the same constraint
- PDB violations cause drain to hang (not fail)
- Single-replica deployments with PDB = undrain-able nodes

## WHY IT MATTERS:
Misconfigured PDBs cause:
- Blocked cluster upgrades
- Failed node maintenance
- Stuck spot instance replacements

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-013

Tomorrow: ResourceQuota limits that prevent deployments.

---

# Killercoda Lab Instructions

## Step 1: Deploy a 2-replica application

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: critical-app
  template:
    metadata:
      labels:
        app: critical-app
    spec:
      containers:
      - name: app
        image: nginx
        ports:
        - containerPort: 80
EOF

```

**Check pod placement:**

```bash
kubectl get pods -l app=critical-app -o wide

```

Note which nodes the pods are on.

Step 2: Create an overly strict PDB

```bash
cat <<EOF | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-app-pdb
spec:
  minAvailable: 2  # All replicas must be available!
  selector:
    matchLabels:
      app: critical-app
EOF

```

**Check PDB status:**

```bash
kubectl get pdb critical-app-pdb

```

ALLOWED DISRUPTIONS: 0 (no pods can be evicted!)

Step 3: Try to drain a node

Get a node with a critical-app pod:

```bash
NODE=$(kubectl get pods -l app=critical-app -o jsonpath='{.items[0].spec.nodeName}')
echo "Draining node: $NODE"

```

**Attempt drain:**

```bash
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data --timeout=30s

```

Watch it hang! After 30 seconds it times out with error about PDB.

Step 4: Check why drain is blocked

```bash
kubectl get pdb critical-app-pdb -o yaml

```

Look at `status.disruptionsAllowed: 0`

```bash
kubectl describe pdb critical-app-pdb

```

Shows which pods are protected.

Step 5: Fix the PDB to allow 1 disruption

```bash
cat <<EOF | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-app-pdb
spec:
  maxUnavailable: 1  # Allow 1 pod to be down
  selector:
    matchLabels:
      app: critical-app
EOF

```

**Check new status:**

```bash
kubectl get pdb critical-app-pdb

```

ALLOWED DISRUPTIONS: 1

Step 6: Drain should work now

```bash
# Uncordon first if previous drain partially worked
kubectl uncordon $NODE

# Try drain again
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data --timeout=60s

```

This time it succeeds! One pod is evicted and rescheduled.

Step 7: Uncordon the node

```bash
kubectl uncordon $NODE

```

Step 8: Test percentage-based PDB

```bash
# Scale up deployment
kubectl scale deployment critical-app --replicas=5

cat <<EOF | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-app-pdb
spec:
  minAvailable: 60%  # At least 60% must stay up
  selector:
    matchLabels:
      app: critical-app
EOF

```

**Check allowed disruptions:**

```bash
kubectl get pdb critical-app-pdb

```

With 5 replicas, 60% = 3 must stay up, so 2 can be disrupted.

Step 9: Single-replica PDB trap

```bash
kubectl scale deployment critical-app --replicas=1

cat <<EOF | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: single-replica-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: critical-app
EOF

```

**Check:**

```bash
kubectl get pdb single-replica-pdb

```

ALLOWED DISRUPTIONS: 0 - node can never be drained!

This is a common misconfiguration for single-replica deployments.

Step 10: PDB with unhealthy pod handling

```bash
kubectl scale deployment critical-app --replicas=3

cat <<EOF | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: unhealthy-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: critical-app
  unhealthyPodEvictionPolicy: AlwaysAllow  # K8s 1.26+
EOF

```

With `AlwaysAllow`, unhealthy pods don't count against the budget and can always be evicted.

Step 11: Multiple PDBs conflict

```bash
cat <<EOF | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: pdb-conflict
spec:
  maxUnavailable: 0  # Zero tolerance!
  selector:
    matchLabels:
      app: critical-app
EOF

```

Now two PDBs select the same pods. The most restrictive wins!

```bash
kubectl get pdb

```

Even though critical-app-pdb allows 1 disruption, pdb-conflict blocks all.

Step 12: Voluntary vs Involuntary disruptions

**PDBs block (voluntary):**

- kubectl drain
- Cluster autoscaler scale-down
- kubectl delete pod (with eviction API)
- Rolling updates (respects PDB)

**PDBs don't block (involuntary):**

- Node crash
- OOM kills
- Direct pod deletion (kubectl delete pod --grace-period=0)
- Resource quota evictions

```bash
# This bypasses PDB!
POD=$(kubectl get pods -l app=critical-app -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod $POD --grace-period=0 --force

```

Key Observations

✅ **minAvailable vs maxUnavailable** - two ways to express availability requirements
✅ **Percentages** - useful for variable replica counts
✅ **Single replica trap** - minAvailable: 1 with 1 replica = never drain-able
✅ **Multiple PDBs** - most restrictive wins
✅ **Voluntary only** - PDBs don't protect against crashes

Production Patterns

**Stateless web app:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  maxUnavailable: 25%
  selector:
    matchLabels:
      app: web

```

**Database (strict):**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
spec:
  minAvailable: 2  # Quorum for HA
  selector:
    matchLabels:
      app: postgres

```

**Batch jobs (allow full disruption):**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: batch-pdb
spec:
  maxUnavailable: 100%  # Jobs can be restarted
  selector:
    matchLabels:
      app: batch-processor

```

Cleanup

```bash
kubectl uncordon $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}')
kubectl delete pdb critical-app-pdb single-replica-pdb unhealthy-pdb pdb-conflict 2>/dev/null
kubectl delete deployment critical-app 2>/dev/null

```

---

Next: Day 14 - ResourceQuota Limits