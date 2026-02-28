## Day 83: Stateful Application Failures - When State Goes Wrong

### Email Copy

**Subject:** Day 83: Stateful Application Failures - When StatefulSets Break

THE IDEA:
Deploy stateful applications and watch them fail. Pods lose data, volumes don't attach,
ordering breaks, network identities conflict, and database clusters split-brain.

THE SETUP:
Deploy StatefulSets with broken volume claims, test pod deletion without graceful
shutdown, break headless services, and discover what fails when state matters.

WHAT I LEARNED:
- StatefulSet pods have stable network identity
- Volumes must exist before pod starts
- Pod deletion waits for graceful termination
- Headless service required for DNS
- ordinal index determines pod name
- OnDelete update strategy prevents auto-updates

WHY IT MATTERS:
Stateful failures cause:
- Data loss when volumes don't attach
- Split-brain in distributed databases
- Cluster quorum failures
- Permanent data corruption
- Recovery requiring manual intervention

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-083

Tomorrow: Ingress controller failures and load balancing issues.

---
Chad
Kube Daily - Week 13: Stateful Systems & Infrastructure


---

## Killercoda Lab Instructions

## Step 1: Deploy basic StatefulSet

```bash
# Create StatefulSet with persistent storage
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  clusterIP: None  # Headless service
  selector:
    app: nginx-stateful
  ports:
  - port: 80
    name: web
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: nginx-headless
  replicas: 3
  selector:
    matchLabels:
      app: nginx-stateful
  template:
    metadata:
      labels:
        app: nginx-stateful
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
          name: web
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

kubectl wait --for=condition=Ready pod -l app=nginx-stateful --timeout=120s
```

```bash
# Check stable pod names
kubectl get pods -l app=nginx-stateful
echo ""
echo "Notice: web-0, web-1, web-2 (ordered)"
```

## Step 2: Test pod ordering

```bash
# StatefulSets create pods sequentially
echo "Pod creation order:"
kubectl get events --sort-by='.lastTimestamp' | grep "web-" | grep Created

echo ""
echo "StatefulSet guarantees:"
echo "- web-0 created first"
echo "- web-1 created after web-0 is Running"
echo "- web-2 created after web-1 is Running"
echo "- Sequential, ordered deployment"
```

## Step 3: Test stable network identity

```bash
# Each pod gets DNS entry
for i in 0 1 2; do
  echo "Testing DNS for web-$i:"
  kubectl run dns-test-$i --rm -i --image=busybox --restart=Never -- \
    nslookup web-$i.nginx-headless.default.svc.cluster.local
done

echo ""
echo "Stable DNS format:"
echo "pod-name.service-name.namespace.svc.cluster.local"
```

## Step 4: Test volume attachment failure

```bash
# Create StatefulSet with non-existent StorageClass
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: broken-storage
spec:
  serviceName: broken-svc
  replicas: 1
  selector:
    matchLabels:
      app: broken
  template:
    metadata:
      labels:
        app: broken
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
      storageClassName: nonexistent-class  # Doesn't exist!
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
EOF

sleep 20
```

```bash
# Pod stuck waiting for volume
kubectl get pods -l app=broken
kubectl get pvc -l app=broken

echo ""
echo "Pod stuck: Volume provisioning failed"
kubectl describe pod -l app=broken | grep -A 5 "Events:"
```

## Step 5: Test pod deletion without graceful shutdown

```bash
# Write data to pod
kubectl exec web-0 -- sh -c 'echo "Important data" > /usr/share/nginx/html/data.txt'
```

```bash
# Verify data exists
kubectl exec web-0 -- cat /usr/share/nginx/html/data.txt
```

```bash
# Force delete pod (no graceful shutdown)
kubectl delete pod web-0 --grace-period=0 --force

sleep 10
```

```bash
# Check if new pod has data (should persist via PVC)
kubectl wait --for=condition=Ready pod web-0 --timeout=120s
kubectl exec web-0 -- cat /usr/share/nginx/html/data.txt

echo ""
echo "Data persisted because PVC reattached to new pod"
```

## Step 6: Test headless service requirement

```bash
# Delete headless service
kubectl delete service nginx-headless

sleep 5
```

```bash
# DNS resolution fails
kubectl run dns-fail-test --rm -i --image=busybox --restart=Never -- \
  nslookup web-0.nginx-headless.default.svc.cluster.local 2>&1 || echo "DNS resolution failed!"

echo ""
echo "Without headless service:"
echo "- Pod DNS names don't resolve"
echo "- Stable network identity broken"
echo "- Database cluster can't communicate"
```

## Step 7: Test StatefulSet scaling

```bash
# Recreate headless service
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  clusterIP: None
  selector:
    app: nginx-stateful
  ports:
  - port: 80
EOF
```

```bash
# Scale down
kubectl scale statefulset web --replicas=1

sleep 20
```

```bash
# Check pod deletion order
kubectl get pods -l app=nginx-stateful

echo ""
echo "Scale down: web-2, then web-1 deleted (reverse order)"
echo "web-0 remains (highest to lowest ordinal)"
```

```bash
# Check PVCs persist
kubectl get pvc
echo ""
echo "PVCs for web-1 and web-2 still exist (not deleted)"
```

## Step 8: Test volume claim deletion

```bash
# Delete StatefulSet
kubectl delete statefulset web --cascade=orphan

sleep 5
```

```bash
# Pods remain (orphaned)
kubectl get pods -l app=nginx-stateful
```

```bash
# PVCs remain
kubectl get pvc

echo ""
echo "StatefulSet deleted but:"
echo "- Pods still running (orphaned)"
echo "- PVCs still exist"
echo "- Manual cleanup required"
```

```bash
# Cleanup orphaned pods
kubectl delete pods -l app=nginx-stateful
```

## Step 9: Test split-brain scenario

```bash
# Deploy etcd cluster
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: etcd
spec:
  clusterIP: None
  selector:
    app: etcd
  ports:
  - port: 2379
    name: client
  - port: 2380
    name: peer
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: etcd
spec:
  serviceName: etcd
  replicas: 3
  selector:
    matchLabels:
      app: etcd
  template:
    metadata:
      labels:
        app: etcd
    spec:
      containers:
      - name: etcd
        image: quay.io/coreos/etcd:v3.5.0
        env:
        - name: ETCD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: ETCD_INITIAL_CLUSTER
          value: "etcd-0=http://etcd-0.etcd:2380,etcd-1=http://etcd-1.etcd:2380,etcd-2=http://etcd-2.etcd:2380"
        - name: ETCD_INITIAL_CLUSTER_STATE
          value: "new"
        - name: ETCD_LISTEN_PEER_URLS
          value: "http://0.0.0.0:2380"
        - name: ETCD_LISTEN_CLIENT_URLS
          value: "http://0.0.0.0:2379"
        - name: ETCD_ADVERTISE_CLIENT_URLS
          value: "http://\$(ETCD_NAME).etcd:2379"
        - name: ETCD_INITIAL_ADVERTISE_PEER_URLS
          value: "http://\$(ETCD_NAME).etcd:2380"
        ports:
        - containerPort: 2379
          name: client
        - containerPort: 2380
          name: peer
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
EOF

kubectl wait --for=condition=Ready pod -l app=etcd --timeout=120s

echo "etcd cluster deployed"
echo ""
echo "Split-brain risk:"
echo "- If network partitions cluster"
echo "- Each partition might elect leader"
echo "- Data divergence"
echo "- Quorum loss"
```

## Step 10: Test parallel pod management

```bash
# StatefulSet with parallel pod management
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: parallel-web
spec:
  serviceName: parallel-svc
  replicas: 3
  podManagementPolicy: Parallel  # All pods created at once
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
EOF

sleep 30

kubectl get events --sort-by='.lastTimestamp' | grep "parallel-web" | grep Created | head -5

echo ""
echo "Parallel pod management:"
echo "- All pods created simultaneously"
echo "- No ordering guarantee"
echo "- Faster startup"
echo "- Use for stateless workloads in StatefulSet"
```

## Step 11: Test OnDelete update strategy

```bash
# Create StatefulSet with OnDelete
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: ondelete-web
spec:
  serviceName: ondelete-svc
  replicas: 2
  updateStrategy:
    type: OnDelete  # Manual updates only
  selector:
    matchLabels:
      app: ondelete
  template:
    metadata:
      labels:
        app: ondelete
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
EOF

kubectl wait --for=condition=Ready pod -l app=ondelete --timeout=60s
```

```bash
# Update image
kubectl patch statefulset ondelete-web -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.22"}]}}}}'

sleep 10
```

```bash
# Pods not updated automatically
kubectl get pods -l app=ondelete -o jsonpath='{.items[*].spec.containers[0].image}'
echo ""
echo "Pods still running nginx:1.21 (OnDelete requires manual pod deletion)"
```

```bash
# Delete one pod to trigger update
kubectl delete pod ondelete-web-0

kubectl wait --for=condition=Ready pod ondelete-web-0 --timeout=60s

kubectl get pod ondelete-web-0 -o jsonpath='{.spec.containers[0].image}'
echo " (updated)"
kubectl get pod ondelete-web-1 -o jsonpath='{.spec.containers[0].image}'
echo " (not updated)"
```

## Step 12: Test RollingUpdate with partition

```bash
# StatefulSet with partition
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: partition-web
spec:
  serviceName: partition-svc
  replicas: 4
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2  # Only update pods >= ordinal 2
  selector:
    matchLabels:
      app: partition
  template:
    metadata:
      labels:
        app: partition
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
EOF

kubectl wait --for=condition=Ready pod -l app=partition --timeout=90s
```

```bash
# Update image
kubectl patch statefulset partition-web -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.22"}]}}}}'

sleep 30
```

```bash
# Check which pods updated
echo "Pod images after update with partition=2:"
for i in 0 1 2 3; do
  IMAGE=$(kubectl get pod partition-web-$i -o jsonpath='{.spec.containers[0].image}')
  echo "partition-web-$i: $IMAGE"
done

echo ""
echo "Only pods 2 and 3 updated (ordinal >= partition)"
```

## Step 13: Test PVC expansion

```bash
# Check if storage class supports expansion
kubectl get storageclass -o jsonpath='{.items[*].allowVolumeExpansion}'
echo ""
```

```bash
# Try to expand PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-web-0
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 2Gi  # Increased from 1Gi
EOF

sleep 10

kubectl get pvc data-web-0

echo ""
echo "PVC expansion:"
echo "- Requires storage class with allowVolumeExpansion: true"
echo "- Some volume types require pod restart"
echo "- Can fail if storage backend doesn't support it"
```

## Step 14: Test graceful termination

```bash
# Deploy with long grace period
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: graceful-web
spec:
  serviceName: graceful-svc
  replicas: 1
  selector:
    matchLabels:
      app: graceful
  template:
    metadata:
      labels:
        app: graceful
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: app
        image: busybox
        command:
        - sh
        - -c
        - |
          trap 'echo "SIGTERM received, cleaning up..."; sleep 30; echo "Cleanup done"' TERM
          while true; do sleep 1; done
EOF

kubectl wait --for=condition=Ready pod -l app=graceful --timeout=60s
```

```bash
# Delete pod and watch
kubectl delete pod -l app=graceful &
DELETE_PID=$!

sleep 5
```

```bash
# Pod in Terminating state
kubectl get pods -l app=graceful

sleep 35
```

```bash
# Check if still terminating
kubectl get pods -l app=graceful 2>&1 || echo "Pod terminated gracefully"

wait $DELETE_PID 2>/dev/null
```

## Step 15: StatefulSet troubleshooting guide

```bash
cat > /tmp/statefulset-troubleshooting.md << 'EOF'
# StatefulSet Troubleshooting Guide

## Common Issues

### Pod Stuck in Pending
kubectl describe pod <pod-name>
kubectl get pvc
kubectl get events --sort-by='.lastTimestamp'

Causes: PVC can't be provisioned, no storage class, volume still attached to old pod

### CrashLoopBackOff on pod-N (not pod-0)
kubectl logs <pod-name>
kubectl logs <pod-name> --previous

Causes: Config depends on pod-0 being ready, wrong initial cluster config

### DNS Resolution Fails
kubectl get svc
kubectl run dns-test --rm -i --image=busybox -- nslookup <service-name>

Causes: Headless service missing, service selector mismatch, CoreDNS issues

### Data Loss After Pod Restart
kubectl get pvc
kubectl describe pvc <pvc-name>

Causes: volumeClaimTemplates missing, emptyDir used instead of PVC

## Best Practices

1. Always use headless service (clusterIP: None)
2. Set terminationGracePeriodSeconds for graceful shutdown
3. Use podAntiAffinity for HA deployments
4. Set appropriate updateStrategy (RollingUpdate with partition for canary)
5. Monitor PVC status separately from pod status
6. Test backup/restore procedures before production
EOF

cat /tmp/statefulset-troubleshooting.md
```

## Key Observations

✅ **Ordered deployment** - pods created sequentially
✅ **Stable identity** - pod names and DNS persist
✅ **Persistent storage** - PVCs survive pod deletion
✅ **Headless service** - required for pod DNS
✅ **Graceful termination** - critical for data safety
✅ **Update strategies** - OnDelete vs RollingUpdate

## Production Patterns

**Production-ready StatefulSet:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: ssd
      resources:
        requests:
          storage: 50Gi
```

## Cleanup

```bash
kubectl delete statefulset web etcd parallel-web ondelete-web partition-web graceful-web broken-storage 2>/dev/null
kubectl delete service nginx-headless etcd parallel-svc ondelete-svc partition-svc graceful-svc broken-svc 2>/dev/null
kubectl delete pvc --all 2>/dev/null
rm -f /tmp/*.md
```

---
Next: Day 84 - Ingress Controller Failures and Load Balancing Issues
