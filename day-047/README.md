## Day 47: StatefulSets - Orphaned Volume Nightmare

## THE IDEA:
Scale a StatefulSet up and down and discover orphaned PersistentVolumeClaims
that never get deleted. Learn when volumes persist vs delete.

## THE SETUP:
Deploy StatefulSet with volumeClaimTemplates, scale up/down, delete pods, and
observe PVC behavior. Test cascade deletion and manual cleanup.

## WHAT I LEARNED:
- Scaling down StatefulSet doesn't delete PVCs (by design!)
- Deleting StatefulSet doesn't delete PVCs (unless cascade=foreground)
- Each pod gets its own PVC that follows naming convention
- Scaling up reattaches to existing PVCs
- Manual PVC cleanup required to reclaim storage

## WHY IT MATTERS:
Orphaned PVCs cause:
- Wasted storage costs from unused volumes
- Cluster quota exhaustion
- Data retention compliance issues

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-047

Week 7 Complete! You've mastered observability and debugging!


---

# Killercoda Lab Instructions


## Step 1: Create StatefulSet with PVCs

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
  name: web
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
        ports:
        - containerPort: 80
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
EOF
```

**Check resources:**
```bash
kubectl get statefulset web
kubectl get pods -l app=stateful
kubectl get pvc
```

3 pods created, 3 PVCs created (data-web-0, data-web-1, data-web-2)

## Step 2: Write data to PVCs

```bash
for i in {0..2}; do
  kubectl exec web-$i -- sh -c "echo 'Data from pod $i' > /usr/share/nginx/html/index.html"
done

# Verify
kubectl exec web-0 -- cat /usr/share/nginx/html/index.html
```

## Step 3: Scale down (PVCs remain!)

```bash
kubectl scale statefulset web --replicas=1

# Check pods and PVCs
kubectl get pods -l app=stateful
kubectl get pvc
```

Only web-0 running, but ALL 3 PVCs still exist!

## Step 4: Scale back up (reattaches to existing PVCs)

```bash
kubectl scale statefulset web --replicas=3

# Wait for pods
kubectl wait --for=condition=Ready pods -l app=stateful --timeout=60s

# Check data persisted
kubectl exec web-1 -- cat /usr/share/nginx/html/index.html
kubectl exec web-2 -- cat /usr/share/nginx/html/index.html
```

Data still there! Scaling up reused existing PVCs.

## Step 5: Delete specific pod (PVC remains)

```bash
kubectl delete pod web-2

# Check PVCs
kubectl get pvc
```

Pod recreated, PVC data-web-2 still exists!

## Step 6: Delete StatefulSet (PVCs still remain!)

```bash
kubectl delete statefulset web

# Check resources
kubectl get statefulset web 2>&1 || echo "StatefulSet deleted"
kubectl get pods -l app=stateful
kubectl get pvc
```

StatefulSet and pods gone, but PVCs still exist!

## Step 7: Manually clean up orphaned PVCs

```bash
kubectl delete pvc data-web-0 data-web-1 data-web-2

# Verify
kubectl get pvc
kubectl get pv
```

## Step 8: Test cascade deletion

```bash
# Recreate StatefulSet
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: stateful-svc
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
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
EOF

kubectl wait --for=condition=Ready pods -l app=stateful --timeout=60s

# Delete with cascade (still doesn't delete PVCs!)
kubectl delete statefulset web --cascade=foreground

kubectl get pvc
```

Even cascade deletion doesn't remove PVCs!

## Step 9: Test PVC retention policy (K8s 1.27+)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-with-retention
spec:
  serviceName: stateful-svc
  replicas: 2
  selector:
    matchLabels:
      app: retention-test
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Delete  # Delete PVCs when StatefulSet deleted
    whenScaled: Retain   # Keep PVCs when scaling down
  template:
    metadata:
      labels:
        app: retention-test
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
EOF
```

**Test deletion:**
```bash
kubectl wait --for=condition=Ready pods -l app=retention-test --timeout=60s
kubectl get pvc

kubectl delete statefulset web-with-retention
sleep 5
kubectl get pvc
```

PVCs automatically deleted! (if K8s 1.27+)

## Step 10: Test PVC expansion

```bash
# Recreate basic StatefulSet
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: stateful-svc
  replicas: 1
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
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
EOF

kubectl wait --for=condition=Ready pods -l app=stateful --timeout=60s

# Try to expand PVC
kubectl patch pvc data-web-0 -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'

# Check status
kubectl get pvc data-web-0 -o jsonpath='{.status.capacity.storage}'
```

## Step 11: Test parallel pod management

```bash
kubectl delete statefulset web
kubectl delete pvc --all

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: parallel-web
spec:
  serviceName: stateful-svc
  replicas: 3
  podManagementPolicy: Parallel  # Start all pods at once
  selector:
    matchLabels:
      app: parallel
  template:
    metadata:
      labels:
        app: parallel
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
EOF
```

**Watch creation:**
```bash
kubectl get pods -l app=parallel -w
```

All pods start simultaneously!

## Step 12: Test StatefulSet update strategies

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rolling-update
spec:
  serviceName: stateful-svc
  replicas: 3
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2  # Only update pods >= 2
  selector:
    matchLabels:
      app: rolling
  template:
    metadata:
      labels:
        app: rolling
        version: v1
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
EOF

kubectl wait --for=condition=Ready pods -l app=rolling --timeout=120s

# Update image
kubectl patch statefulset rolling-update -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.26"}]}}}}'

# Check versions
kubectl get pods -l app=rolling -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
```

Only pod-2 updated (partition=2)!

## Step 13: Find and clean orphaned PVCs

```bash
# List all PVCs
kubectl get pvc -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,CLAIM:.spec.volumeName

# Find PVCs not attached to pods
kubectl get pvc -o json | jq -r '.items[] | select(.metadata.ownerReferences == null) | .metadata.name'

# Script to identify orphans
cat > /tmp/find-orphan-pvcs.sh << 'EOF'
#!/bin/bash
for pvc in $(kubectl get pvc -o name); do
  pvc_name=${pvc#*/}
  # Check if any pod is using it
  if ! kubectl get pods -o json | jq -e ".items[] | .spec.volumes[]? | .persistentVolumeClaim? | select(.claimName == \"$pvc_name\")" > /dev/null 2>&1; then
    echo "Orphaned: $pvc_name"
  fi
done
EOF

chmod +x /tmp/find-orphan-pvcs.sh
/tmp/find-orphan-pvcs.sh
```

## Step 14: Test StatefulSet with init containers

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: init-web
spec:
  serviceName: stateful-svc
  replicas: 2
  selector:
    matchLabels:
      app: init-test
  template:
    metadata:
      labels:
        app: init-test
    spec:
      initContainers:
      - name: setup
        image: busybox
        command: ['sh', '-c', 'echo "Initialized by init container" > /data/init.txt']
        volumeMounts:
        - name: data
          mountPath: /data
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
EOF

# Verify init worked
kubectl wait --for=condition=Ready pods -l app=init-test --timeout=60s
kubectl exec init-web-0 -- cat /usr/share/nginx/html/init.txt
```

## Step 15: Monitor PVC usage and cleanup

```bash
# Check PV reclaim policy
kubectl get pv -o custom-columns=NAME:.metadata.name,CLAIM:.spec.claimRef.name,RECLAIM:.spec.persistentVolumeReclaimPolicy,STATUS:.status.phase

# Delete all test resources
kubectl delete statefulset --all
kubectl delete service stateful-svc

# List remaining PVCs
kubectl get pvc

# Cleanup all PVCs
kubectl delete pvc --all

# Check PVs are released
kubectl get pv
```

## Key Observations

✅ **Scale down** - doesn't delete PVCs (by design)
✅ **Delete StatefulSet** - doesn't delete PVCs (unless retention policy set)
✅ **Scale up** - reattaches to existing PVCs
✅ **Naming convention** - {pvc-name}-{statefulset-name}-{ordinal}
✅ **Manual cleanup** - required to reclaim storage
✅ **Retention policy** - K8s 1.27+ can auto-delete PVCs

## Production Patterns

**StatefulSet with auto-cleanup (K8s 1.27+):**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: db-svc
  replicas: 3
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Delete  # Auto-delete on StatefulSet deletion
    whenScaled: Retain   # Keep when scaling down (data safety)
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

**Staged rollout with partition:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: app
spec:
  replicas: 10
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 8  # Only update pods 8-9 (canary)
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:v2
```

**Automated PVC cleanup script:**
```bash
#!/bin/bash
# cleanup-orphaned-pvcs.sh

NAMESPACE=${1:-default}
DRY_RUN=${2:-true}

echo "Finding orphaned PVCs in namespace: $NAMESPACE"

# Get all PVCs
for pvc in $(kubectl get pvc -n $NAMESPACE -o name); do
  pvc_name=${pvc#persistentvolumeclaim/}

  # Check if PVC is used by any pod
  if ! kubectl get pods -n $NAMESPACE -o json | \
       jq -e ".items[] | .spec.volumes[]? | \
              .persistentVolumeClaim? | \
              select(.claimName == \"$pvc_name\")" > /dev/null 2>&1; then

    # Check if it's part of StatefulSet pattern
    if [[ $pvc_name =~ ^.*-[0-9]+$ ]]; then
      base_name=${pvc_name%-*}
      ordinal=${pvc_name##*-}

      # Check if StatefulSet exists
      if ! kubectl get statefulset -n $NAMESPACE -o json | \
           jq -e ".items[] | select(.spec.volumeClaimTemplates[]?.metadata.name == \"${base_name}\")" > /dev/null 2>&1; then
        echo "Orphaned PVC: $pvc_name (StatefulSet likely deleted)"

        if [ "$DRY_RUN" != "true" ]; then
          kubectl delete pvc -n $NAMESPACE $pvc_name
          echo "  Deleted!"
        else
          echo "  [DRY RUN] Would delete"
        fi
      fi
    else
      echo "Orphaned PVC: $pvc_name (not attached to any pod)"

      if [ "$DRY_RUN" != "true" ]; then
        kubectl delete pvc -n $NAMESPACE $pvc_name
        echo "  Deleted!"
      else
        echo "  [DRY RUN] Would delete"
      fi
    fi
  fi
done

echo "Cleanup complete!"
```

**Usage:**
```bash
# Dry run
./cleanup-orphaned-pvcs.sh default true

# Actually delete
./cleanup-orphaned-pvcs.sh default false
```

**Monitoring PVC costs:**
```bash
# Calculate total storage claimed
kubectl get pvc -A -o json | jq -r '
  .items |
  group_by(.metadata.namespace) |
  map({
    namespace: .[0].metadata.namespace,
    total_gb: (map(.spec.resources.requests.storage |
                   gsub("Gi"; "") | tonumber) | add)
  }) |
  .[] |
  "\(.namespace): \(.total_gb)Gi"
'
```

**Alert on orphaned PVCs:**
```yaml
# Prometheus alert
- alert: OrphanedPVCs
  expr: |
    count(kube_persistentvolumeclaim_info)
    -
    count(kube_pod_spec_volumes_persistentvolumeclaims_info)
    > 5
  for: 1h
  annotations:
    summary: "More than 5 orphaned PVCs detected"
```

## Cleanup

```bash
kubectl delete statefulset --all
kubectl delete service stateful-svc 2>/dev/null
kubectl delete pvc --all
rm -f /tmp/find-orphan-pvcs.sh
```

---
