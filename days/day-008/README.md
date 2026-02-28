# Day 8: PVC Binding Failures - Storage That Never Arrives

## THE IDEA:
Create a PersistentVolumeClaim requesting a StorageClass that doesn't exist.
Watch the pod hang in Pending forever while Kubernetes silently waits for
storage that will never come.

## THE SETUP:
Deploy a pod with a PVC referencing a nonexistent StorageClass. Then fix it by
creating the right StorageClass, and explore WaitForFirstConsumer binding mode.

## WHAT I LEARNED:
- PVCs in Pending state don't generate obvious errors
- WaitForFirstConsumer delays binding until pod is scheduled
- Immediate binding can fail in multi-zone clusters
- StorageClass reclaim policies determine what happens to data after PVC deletion

## WHY IT MATTERS:
Storage failures are silent killers:
- Pods stuck in Pending with no clear error message
- Data loss from wrong reclaimPolicy (Delete vs Retain)
- Zone mismatches leaving pods unschedulable

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-008

---

## Killercoda Lab Instructions


### Step 1: Check available StorageClasses

```bash
kubectl get storageclass

```

Note the available classes. We'll request one that doesn't exist.

### Step 2: Create a PVC with nonexistent StorageClass

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: broken-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: super-fast-nvme  # Doesn't exist!
  resources:
    requests:
      storage: 1Gi
EOF

```

**Check PVC status:**

```bash
kubectl get pvc broken-pvc

```

Status: Pending. No error, just... waiting.

### Step 3: Deploy a pod using this PVC

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: storage-app
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
      claimName: broken-pvc
EOF

```

**Check pod status:**

```bash
kubectl get pod storage-app

```

Status: Pending. Let's find out why.

### Step 4: Diagnose the issue

```bash
kubectl describe pod storage-app | grep -A 5 Events

```

You'll see: "persistentvolumeclaim 'broken-pvc' not found" or "unbound immediate PersistentVolumeClaims"

```bash
kubectl describe pvc broken-pvc | grep -A 5 Events

```

Look for: "[storageclass.storage.k8s.io](http://storageclass.storage.k8s.io/) 'super-fast-nvme' not found"

### Step 5: Create the missing StorageClass (simulated)

```bash
# First, get the default provisioner
DEFAULT_PROVISIONER=$(kubectl get storageclass -o jsonpath='{.items[0].provisioner}')
echo "Using provisioner: $DEFAULT_PROVISIONER"

cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: super-fast-nvme
provisioner: $DEFAULT_PROVISIONER
reclaimPolicy: Delete
volumeBindingMode: Immediate
EOF

```

**Watch PVC bind:**

```bash
kubectl get pvc broken-pvc -w

```

Status changes: Pending → Bound

**Check pod now:**

```bash
kubectl get pod storage-app

```

Should transition to Running!

### Step 6: Test WaitForFirstConsumer binding mode

```bash
kubectl delete pod storage-app
kubectl delete pvc broken-pvc

cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: delayed-binding
provisioner: $DEFAULT_PROVISIONER
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: delayed-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: delayed-binding
  resources:
    requests:
      storage: 1Gi
EOF

```

**Check PVC status:**

```bash
kubectl get pvc delayed-pvc

```

Status: Pending (but this is expected!)

```bash
kubectl describe pvc delayed-pvc | grep -A 3 Events

```

Message: "waiting for first consumer to be created before binding"

### Step 7: Create a consumer pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: consumer-app
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
      claimName: delayed-pvc
EOF

```

**Watch both resources:**

```bash
kubectl get pvc,pod -w

```

PVC binds only after pod is scheduled to a specific node!

### Step 8: Test reclaim policies

```bash
# Create PVC with Retain policy
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: retain-storage
provisioner: $DEFAULT_PROVISIONER
reclaimPolicy: Retain
volumeBindingMode: Immediate
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: retain-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: retain-storage
  resources:
    requests:
      storage: 1Gi
EOF

```

**Wait for binding, then delete the PVC:**

```bash
kubectl get pvc retain-pvc -w  # Wait until Bound
kubectl delete pvc retain-pvc

```

**Check PV status:**

```bash
kubectl get pv

```

PV status: Released (not deleted!). Data is preserved but PV can't be reused without manual intervention.

### Step 9: Access mode conflicts

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-pvc
spec:
  accessModes:
    - ReadWriteMany  # Not supported by all provisioners!
  storageClassName: super-fast-nvme
  resources:
    requests:
      storage: 1Gi
EOF

```

**Check status:**

```bash
kubectl describe pvc shared-pvc | grep -A 5 Events

```

Many provisioners don't support ReadWriteMany. You might see provisioning failures.

Key Observations

✅ **Pending PVCs are silent** - no automatic errors, just waiting
✅ **WaitForFirstConsumer** - prevents zone mismatch issues in multi-zone clusters
✅ **Retain vs Delete** - controls data lifecycle after PVC deletion
✅ **Access modes matter** - RWX requires specific storage backends (NFS, EFS, etc.)

Production Patterns

**Multi-zone safe storage:**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: zone-aware
provisioner: kubernetes.io/aws-ebs
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
- matchLabelExpressions:
  - key: topology.kubernetes.io/zone
    values:
    - us-east-1a
    - us-east-1b

```

**Expand existing PVC (if supported):**

```yaml
spec:
  resources:
    requests:
      storage: 10Gi  # Increase from original size

```

**StatefulSet with volumeClaimTemplates:**

```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: ["ReadWriteOnce"]
    storageClassName: fast-ssd
    resources:
      requests:
        storage: 100Gi

```

Cleanup

```bash
kubectl delete pod storage-app consumer-app 2>/dev/null
kubectl delete pvc broken-pvc delayed-pvc retain-pvc shared-pvc 2>/dev/null
kubectl delete storageclass super-fast-nvme delayed-binding retain-storage 2>/dev/null
kubectl delete pv --all 2>/dev/null

```

---

Next: Day 9 - ConfigMap Updates That Don't Reach Your Pods