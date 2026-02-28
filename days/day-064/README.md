## Day 64: PV Provisioning Failures - Storage Chaos

### Email Copy

**Subject:** Day 64: PV Provisioning Failures - When Storage Won't Attach

THE IDEA:
Request persistent volumes and watch provisioning fail. PVCs stuck in Pending, pods
can't start due to mount failures, and storage classes misconfigured.

THE SETUP:
Create StorageClasses with wrong provisioners, request volumes that can't be provisioned,
hit volume attach limits, and test what breaks when storage fails.

WHAT I LEARNED:
- Dynamic provisioning requires StorageClass
- CSI driver must be installed and healthy
- Volume attachment limits per node (AWS: 39 EBS volumes)
- PVC stuck Pending if no PV matches or provisioning fails
- Pods in ContainerCreating when volume won't attach

WHY IT MATTERS:
Storage failures cause:
- StatefulSets stuck unable to scale
- Databases can't start (no persistent storage)
- Data loss if using wrong ReclaimPolicy
- Cluster capacity wasted by unattachable volumes

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-064

Tomorrow: Image pull failures and registry issues.

---
Chad


---

## Killercoda Lab Instructions


## Step 1: Check available StorageClasses

```bash
# List storage classes
kubectl get storageclass

# Check default storage class
kubectl get storageclass -o jsonpath='{.items[?(@.metadata.annotations.storageclass\.kubernetes\.io/is-default-class=="true")].metadata.name}'
echo ""
```

## Step 2: Test basic PVC creation

```bash
# Create PVC
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

# Check status
kubectl get pvc test-pvc
kubectl describe pvc test-pvc
```

## Step 3: Create StorageClass with invalid provisioner

```bash
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: broken-storage
provisioner: kubernetes.io/nonexistent
parameters:
  type: invalid
volumeBindingMode: Immediate
EOF
```

## Step 4: Request volume with broken StorageClass

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: broken-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: broken-storage
  resources:
    requests:
      storage: 1Gi
EOF

# Check status (stuck Pending)
sleep 5
kubectl get pvc broken-pvc
kubectl describe pvc broken-pvc | grep -A 10 "Events:"
```

Shows "no such provisioner"!

## Step 5: Deploy pod with PVC

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: broken-pvc
EOF

# Check pod status (stuck ContainerCreating)
sleep 10
kubectl get pod pvc-pod
kubectl describe pod pvc-pod | grep -A 10 "Events:"
```

Pod can't start - waiting for volume!

## Step 6: Test volume with insufficient capacity

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: huge-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Ti  # Unrealistic size
EOF

kubectl get pvc huge-pvc
kubectl describe pvc huge-pvc | grep -A 5 "Events:"
```

## Step 7: Test access mode mismatch

```bash
# Create PV with ReadOnlyMany
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: readonly-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadOnlyMany
  hostPath:
    path: /tmp/readonly
  persistentVolumeReclaimPolicy: Delete
EOF

# Request with ReadWriteOnce (won't match)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mismatch-pvc
spec:
  accessModes:
  - ReadWriteOnce  # Doesn't match PV
  resources:
    requests:
      storage: 1Gi
  storageClassName: ""  # Use manual binding
EOF

kubectl get pvc mismatch-pvc
```

## Step 8: Test storage with wrong volume mode

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: block-pvc
spec:
  accessModes:
  - ReadWriteOnce
  volumeMode: Block  # Raw block device
  resources:
    requests:
      storage: 1Gi
EOF

# Most storage classes expect Filesystem, not Block
kubectl describe pvc block-pvc
```

## Step 9: Test StatefulSet with broken storage

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
  name: broken-statefulset
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
      - name: app
        image: nginx
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: broken-storage  # Broken!
      resources:
        requests:
          storage: 1Gi
EOF

# Check status
sleep 10
kubectl get statefulset broken-statefulset
kubectl get pods -l app=stateful
kubectl get pvc -l app=stateful
```

StatefulSet can't scale - PVCs stuck!

## Step 10: Test volume attachment limits

```bash
# Simulate hitting volume limit (concept)
echo "Volume attachment limits per node:"
echo "- AWS EBS: 39 volumes per instance"
echo "- GCP PD: 127 volumes per node"
echo "- Azure Disk: 64 volumes per node"
echo ""
echo "When limit reached:"
echo "- New PVCs remain Pending"
echo "- Pods stuck in ContainerCreating"
echo "- Error: 'AttachVolume.Attach failed: Maximum number of volumes reached'"
```

## Step 11: Test ReclaimPolicy impact

```bash
# Create PV with Delete policy
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: delete-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete  # Deletes volume when PVC deleted
  hostPath:
    path: /tmp/delete-test
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: delete-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: ""
  volumeName: delete-pv
EOF

# Check binding
kubectl get pv delete-pv
kubectl get pvc delete-pvc

# Delete PVC (PV will be deleted too!)
kubectl delete pvc delete-pvc

sleep 5
kubectl get pv delete-pv 2>&1 || echo "PV deleted (Retain would preserve it)"
```

## Step 12: Test WaitForFirstConsumer binding

```bash
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: wait-for-consumer
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer  # Delays binding
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wait-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: wait-for-consumer
  hostPath:
    path: /tmp/wait
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wait-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: wait-for-consumer
  resources:
    requests:
      storage: 1Gi
EOF

# PVC stays Pending until pod uses it
kubectl get pvc wait-pvc

# Create pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: wait-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: wait-pvc
EOF

# Now PVC binds
sleep 5
kubectl get pvc wait-pvc
```

## Step 13: Test volume expansion (if supported)

```bash
# Check if storage class allows expansion
kubectl get storageclass -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.allowVolumeExpansion}{"\n"}{end}'

# Try to expand PVC
kubectl patch pvc test-pvc -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'

# Check if expansion supported
kubectl describe pvc test-pvc | grep -i "expansion\|resize"
```

## Step 14: Check CSI driver health

```bash
# Check for CSI driver pods
kubectl get pods -A | grep csi

# Check CSI node pods (one per node)
kubectl get pods -A | grep "csi-node\|csi-attacher\|csi-provisioner"

# If CSI driver down, volume operations fail
```

## Step 15: Diagnose storage issues

```bash
cat > /tmp/storage-diagnosis.sh << 'EOF'
#!/bin/bash
echo "=== Storage Diagnosis Report ==="

echo -e "\n1. StorageClasses:"
kubectl get storageclass

echo -e "\n2. Pending PVCs:"
kubectl get pvc -A --field-selector status.phase=Pending

echo -e "\n3. Unbound PVs:"
kubectl get pv --field-selector status.phase=Available

echo -e "\n4. Recent PVC Events:"
kubectl get events -A --sort-by='.lastTimestamp' | grep -i "persistentvolumeclaim\|pvc" | tail -10

echo -e "\n5. Pods Waiting for Volumes:"
kubectl get pods -A -o json | jq -r '
  .items[] |
  select(.status.conditions[]? | .type == "PodScheduled" and .reason == "Unschedulable") |
  "\(.metadata.namespace)/\(.metadata.name): \(.status.conditions[] | select(.type == "PodScheduled").message)"
' | grep -i volume || echo "No pods waiting for volumes"

echo -e "\n6. Volume Attachment Status:"
kubectl get volumeattachment 2>/dev/null | head -10 || echo "VolumeAttachments not available"

echo -e "\n7. Storage Capacity:"
kubectl get pv -o custom-columns=NAME:.metadata.name,CAPACITY:.spec.capacity.storage,STATUS:.status.phase

echo -e "\n8. Common Storage Issues:"
echo "   - Wrong StorageClass: PVC pending"
echo "   - No provisioner: Dynamic provisioning fails"
echo "   - Volume limit reached: Attachments fail"
echo "   - Access mode mismatch: PVC won't bind"
echo "   - CSI driver down: All storage operations fail"
echo "   - Wrong ReclaimPolicy: Data deleted unintentionally"
EOF

chmod +x /tmp/storage-diagnosis.sh
/tmp/storage-diagnosis.sh
```

## Key Observations

✅ **StorageClass required** - for dynamic provisioning
✅ **Provisioner must exist** - or PVC stays Pending
✅ **Access modes must match** - between PV and PVC
✅ **Volume limits exist** - per cloud provider
✅ **ReclaimPolicy matters** - Delete vs Retain
✅ **WaitForFirstConsumer** - delays binding until pod scheduled

## Production Patterns

**Production StorageClass:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  kmsKeyId: arn:aws:kms:us-east-1:123456789:key/...
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Retain  # Don't delete on PVC deletion
```

**StatefulSet with proper storage:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: database
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:15
        volumeMounts:
        - name: pgdata
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: pgdata
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
```

**Storage monitoring:**
```yaml
# Prometheus alerts
- alert: PVCPendingTooLong
  expr: |
    kube_persistentvolumeclaim_status_phase{phase="Pending"} == 1
  for: 10m
  annotations:
    summary: "PVC {{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }} pending > 10min"

- alert: VolumeAttachmentFailed
  expr: |
    kube_pod_status_phase{phase="Pending"} == 1
  for: 15m
  annotations:
    summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} stuck pending"

- alert: StorageNearFull
  expr: |
    (kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes) > 0.9
  annotations:
    summary: "Volume {{ $labels.persistentvolumeclaim }} > 90% full"
```

## Cleanup

```bash
kubectl delete pod pvc-pod wait-pod
kubectl delete pvc test-pvc broken-pvc huge-pvc mismatch-pvc block-pvc wait-pvc delete-pvc 2>/dev/null
kubectl delete pv readonly-pv wait-pv 2>/dev/null
kubectl delete statefulset broken-statefulset
kubectl delete service stateful-svc
kubectl delete storageclass broken-storage wait-for-consumer
rm -f /tmp/storage-diagnosis.sh
```

---
Next: Day 65 - Image Pull Failures and Registry Issues
