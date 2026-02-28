# Day 21: Finalizers - The Deletion Blocker

## THE IDEA:
Add a finalizer to a namespace, then try to delete it. Watch the namespace get 
stuck in Terminating forever. Learn to identify and remove blocking finalizers.

## THE SETUP:
Create resources with custom finalizers, attempt deletion, and discover they're 
stuck. Explore how finalizers protect resources and how to force cleanup.

## WHAT I LEARNED:
- Finalizers are pre-delete hooks that must complete
- Stuck in Terminating = finalizer not removed
- kubectl delete doesn't bypass finalizers (by design)
- Can patch resources to remove finalizers (danger!)

## WHY IT MATTERS:
Finalizer issues cause:
- Namespaces stuck Terminating for hours/days
- Resources can't be deleted during cleanup
- Cluster garbage collection blocked

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-021

Week 3 complete! 21 days of Kubernetes failure mastery unlocked.

---

## Killercoda Lab Instructions

### Step 1: Create a simple resource with finalizer

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-cm
  finalizers:
  - example.com/finalizer
data:
  key: value
EOF

```

### Step 2: Try to delete it

```bash
kubectl delete configmap test-cm

```

**Check status:**

```bash
kubectl get configmap test-cm

```

Still there! Has deletion timestamp but not deleted.

```bash
kubectl get configmap test-cm -o yaml | grep -A 5 metadata

```

Shows:

- `deletionTimestamp` set
- `finalizers` still present

### Step 3: Remove finalizer to allow deletion

```bash
kubectl patch configmap test-cm -p '{"metadata":{"finalizers":[]}}' --type=merge

```

**Verify:**

```bash
kubectl get configmap test-cm

```

Now it's gone! Deletion completed.

### Step 4: Test namespace with finalizer

```bash
kubectl create namespace stuck-namespace

kubectl patch namespace stuck-namespace -p '{"metadata":{"finalizers":["example.com/stuck"]}}'

```

**Try to delete:**

```bash
kubectl delete namespace stuck-namespace

```

**Check status:**

```bash
kubectl get namespace stuck-namespace

```

Status: Terminating (stuck!)

```bash
kubectl get namespace stuck-namespace -o yaml | grep -A 5 status

```

Shows `phase: Terminating`.

### Step 5: List all finalizers on the namespace

```bash
kubectl get namespace stuck-namespace -o jsonpath='{.metadata.finalizers}'
echo ""

```

### Step 6: Remove finalizer to complete deletion

```bash
kubectl patch namespace stuck-namespace -p '{"metadata":{"finalizers":[]}}' --type=merge

```

Namespace immediately deletes!

### Step 7: Test PVC with protection finalizer

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

```

**Check built-in finalizers:**

```bash
kubectl get pvc test-pvc -o jsonpath='{.metadata.finalizers}'
echo ""

```

Shows: `kubernetes.io/pvc-protection` (prevents deletion while in use)

### Step 8: Use PVC in a pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pvc-user
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: test-pvc
EOF

```

**Try to delete PVC while pod is using it:**

```bash
kubectl delete pvc test-pvc

```

**Check status:**

```bash
kubectl get pvc test-pvc

```

Stuck in Terminating! Finalizer protects PVC while in use.

### Step 9: Delete pod to allow PVC deletion

```bash
kubectl delete pod pvc-user

```

**Watch PVC:**

```bash
kubectl get pvc test-pvc -w

```

Once pod is gone, PVC deletion completes!

### Step 10: Custom resource with controller finalizer

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: finalizer-test
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: controlled-resource
  namespace: finalizer-test
  finalizers:
  - operator.example.com/cleanup
data:
  managed: "true"
EOF

```

**Simulate controller failing to process:**

```bash
kubectl delete configmap controlled-resource -n finalizer-test

```

**Resource stuck:**

```bash
kubectl get configmap -n finalizer-test

```

In production, controller would remove finalizer after cleanup.

### Step 11: Inspect deletion timestamp

```bash
kubectl get configmap controlled-resource -n finalizer-test -o jsonpath='{.metadata.deletionTimestamp}'
echo ""

```

Shows when deletion was requested.

### Step 12: Force removal (emergency only!)

```bash
# WARNING: Only use in emergencies!
kubectl patch configmap controlled-resource -n finalizer-test -p '{"metadata":{"finalizers":[]}}' --type=merge

# Or edit directly
kubectl edit configmap controlled-resource -n finalizer-test
# Remove finalizers array manually

```

### Step 13: Test multiple finalizers

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: multi-finalizer
  finalizers:
  - finalizer1.example.com
  - finalizer2.example.com
  - finalizer3.example.com
data:
  test: data
EOF

```

**Delete it:**

```bash
kubectl delete configmap multi-finalizer

```

**Remove finalizers one by one:**

```bash
# Remove first
kubectl patch configmap multi-finalizer -p '{"metadata":{"finalizers":["finalizer2.example.com","finalizer3.example.com"]}}' --type=merge

# Check - still there
kubectl get configmap multi-finalizer

# Remove remaining
kubectl patch configmap multi-finalizer -p '{"metadata":{"finalizers":[]}}' --type=merge

# Now it deletes

```

### Step 14: Common built-in finalizers

```bash
# Check various resources for finalizers
kubectl get namespace default -o jsonpath='{.metadata.finalizers}' && echo ""
kubectl get pv -o jsonpath='{.items[*].metadata.finalizers}' && echo ""
kubectl get svc kubernetes -o jsonpath='{.metadata.finalizers}' && echo ""

```

Common finalizers:

- `kubernetes.io/pvc-protection`
- `kubernetes.io/pv-protection`
- `kubernetes` (for namespaces)

### Step 15: Find all stuck resources in Terminating state

```bash
# Find stuck namespaces
kubectl get namespaces | grep Terminating

# Find all resources with deletion timestamp
kubectl get pods --all-namespaces -o json | jq -r '.items[] | select(.metadata.deletionTimestamp != null) | "\\(.metadata.namespace)/\\(.metadata.name)"'

# Find resources with finalizers
kubectl get configmaps --all-namespaces -o json | jq -r '.items[] | select(.metadata.finalizers != null) | "\\(.metadata.namespace)/\\(.metadata.name): \\(.metadata.finalizers)"'

```

### Step 16: Cleanup stuck namespace (production scenario)

```bash
# Create namespace with resources
kubectl create namespace cleanup-test
kubectl create configmap test1 -n cleanup-test
kubectl create configmap test2 -n cleanup-test --dry-run=client -o yaml | kubectl apply -f - --namespace=cleanup-test
kubectl patch namespace cleanup-test -p '{"metadata":{"finalizers":["stuck.finalizer.com"]}}'

# Delete namespace
kubectl delete namespace cleanup-test

# It's stuck!
kubectl get namespace cleanup-test

# Method 1: Remove finalizer
kubectl patch namespace cleanup-test -p '{"metadata":{"finalizers":[]}}' --type=merge

# Method 2: Use kubectl raw API (alternative)
# kubectl get namespace cleanup-test -o json | jq '.metadata.finalizers = []' | kubectl replace --raw /api/v1/namespaces/cleanup-test/finalize -f -

```

### Key Observations

✅ **Finalizers block deletion** - resource stays until finalizer removed
✅ **deletionTimestamp set** - indicates deletion requested
✅ **Built-in finalizers** - protect resources from premature deletion
✅ **Multiple finalizers** - ALL must be removed
✅ **Force removal** - last resort, can break controllers

### Production Patterns

**Custom controller with finalizer:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: managed-resource
  finalizers:
  - myoperator.example.com/cleanup
data:
  externalResource: "db-instance-123"

```

**Controller cleanup logic:**

```go
// Pseudocode
if resource.DeletionTimestamp != nil {
    // Delete external resources
    deleteExternalDB(resource.Data["externalResource"])

    // Remove finalizer
    removeFinalizer(resource, "myoperator.example.com/cleanup")
    update(resource)
}

```

**PVC protection (automatic):**

```yaml
# Kubernetes automatically adds this
metadata:
  finalizers:
  - kubernetes.io/pvc-protection

```

### Emergency Cleanup Script

```bash
#!/bin/bash
# Use with caution!

NAMESPACE=$1
if [ -z "$NAMESPACE" ]; then
  echo "Usage: $0 <namespace>"
  exit 1
fi

echo "Removing finalizers from namespace: $NAMESPACE"
kubectl patch namespace $NAMESPACE -p '{"metadata":{"finalizers":[]}}' --type=merge

echo "Namespace should now delete"

```

### Cleanup

```bash
kubectl delete namespace finalizer-test stuck-namespace cleanup-test 2>/dev/null
kubectl delete pod pvc-user 2>/dev/null
kubectl delete pvc test-pvc 2>/dev/null
kubectl delete configmap test-cm multi-finalizer 2>/dev/null

```

---

🎉 Week 3 Complete! 21 days of Kubernetes failure modes conquered.