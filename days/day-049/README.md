## Day 49: Backup & Restore - Velero Disasters

### Email Copy

**Subject:** Day 49: Backup & Restore - When Velero Fails You

THE IDEA:
Install Velero for backup/restore and watch it fail from missing volume snapshots,
incomplete restores, and storage provider issues. Learn what gets backed up (and what doesn't).

THE SETUP:
Create backups of namespaces and PVs, simulate disasters, attempt restore, and
discover missing resources, failed volume snapshots, and restore conflicts.

WHAT I LEARNED:
- Not all resources are backed up by default (events, nodes)
- PV snapshots require cloud provider integration
- Restore creates new resources (doesn't update existing)
- Hooks can run pre/post backup for consistency
- Backup location must be accessible from all clusters

WHY IT MATTERS:
Backup failures cause:
- Unrecoverable data loss after disasters
- Incomplete application restores
- RTO violations (slower recovery than planned)

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-049

Tomorrow: Node failures and pod eviction storms.

---
Chad


---

## Killercoda Lab Instructions


## Step 1: Install Velero CLI

```bash
# Download Velero CLI
wget https://github.com/vmware-tanzu/velero/releases/download/v1.12.0/velero-v1.12.0-linux-amd64.tar.gz
tar -xvf velero-v1.12.0-linux-amd64.tar.gz
sudo mv velero-v1.12.0-linux-amd64/velero /usr/local/bin/
velero version --client-only
```

## Step 2: Install Velero in cluster (without cloud provider)

```bash
# Install Velero with filesystem backup location
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket velero-backups \
  --secret-file /dev/null \
  --use-volume-snapshots=false \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000 \
  --use-node-agent \
  --uploader-type=restic

# Wait for pods
kubectl wait --for=condition=Ready pods --all -n velero --timeout=120s
```

**Note:** In production, use real S3/GCS/Azure storage

## Step 3: Deploy MinIO for backup storage (local testing)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: velero
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: velero
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        image: minio/minio:latest
        args: ["server", "/data"]
        env:
        - name: MINIO_ACCESS_KEY
          value: "minio"
        - name: MINIO_SECRET_KEY
          value: "minio123"
        ports:
        - containerPort: 9000
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: velero
spec:
  selector:
    app: minio
  ports:
  - port: 9000
    targetPort: 9000
EOF

kubectl wait --for=condition=Ready pod -l app=minio -n velero --timeout=60s
```

## Step 4: Create test application to backup

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: backup-test
  labels:
    app: test
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  namespace: backup-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
      volumes:
      - name: data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: backup-test
spec:
  selector:
    app: nginx
  ports:
  - port: 80
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: backup-test
data:
  config.json: |
    {"environment": "production"}
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: backup-test
type: Opaque
stringData:
  password: "super-secret-password"
EOF
```

**Write data to pods:**
```bash
for pod in $(kubectl get pods -n backup-test -l app=nginx -o name); do
  kubectl exec -n backup-test $pod -- sh -c 'echo "Important data" > /usr/share/nginx/html/index.html'
done
```

## Step 5: Create backup of namespace

```bash
velero backup create backup-test-ns \
  --include-namespaces backup-test \
  --wait

# Check backup status
velero backup describe backup-test-ns
```

**Check what was backed up:**
```bash
velero backup logs backup-test-ns | head -50
```

## Step 6: Test backup with missing volume snapshots

```bash
# Create app with PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
  namespace: backup-test
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
  namespace: backup-test
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'echo "PVC data" > /data/file.txt; sleep 3600']
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: data-pvc
EOF

kubectl wait --for=condition=Ready pod -n backup-test pvc-pod --timeout=60s

# Backup with PVC
velero backup create backup-with-pvc \
  --include-namespaces backup-test \
  --wait
```

**Check for volume snapshot errors:**
```bash
velero backup describe backup-with-pvc --details
```

PVC backed up but NO volume snapshot (no cloud provider)!

## Step 7: Simulate disaster (delete namespace)

```bash
kubectl delete namespace backup-test

# Verify deletion
kubectl get namespace backup-test 2>&1 || echo "Namespace deleted"
kubectl get pods -n backup-test 2>&1 || echo "Pods gone"
```

## Step 8: Restore from backup

```bash
velero restore create restore-test-1 \
  --from-backup backup-test-ns \
  --wait

# Check restore status
velero restore describe restore-test-1
```

**Verify restoration:**
```bash
kubectl get all -n backup-test
kubectl get configmap -n backup-test
kubectl get secret -n backup-test
```

## Step 9: Check for incomplete restore

```bash
# Check if data was restored
POD=$(kubectl get pods -n backup-test -l app=nginx -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n backup-test $POD -- cat /usr/share/nginx/html/index.html 2>&1 || echo "Data NOT restored (emptyDir)"

# Check PVC pod
kubectl get pod pvc-pod -n backup-test
kubectl exec -n backup-test pvc-pod -- cat /data/file.txt 2>&1 || echo "PVC data lost (no snapshot)"
```

emptyDir data lost! PVC data lost (no volume snapshots)!

## Step 10: Test backup with labels

```bash
# Delete namespace again
kubectl delete namespace backup-test

# Recreate with labels
kubectl create namespace backup-test
kubectl label namespace backup-test backup=enabled

# Create resources with different labels
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: important-pod
  namespace: backup-test
  labels:
    tier: critical
spec:
  containers:
  - name: app
    image: nginx
---
apiVersion: v1
kind: Pod
metadata:
  name: unimportant-pod
  namespace: backup-test
  labels:
    tier: optional
spec:
  containers:
  - name: app
    image: nginx
EOF

# Backup only critical resources
velero backup create selective-backup \
  --selector tier=critical \
  --wait

velero backup describe selective-backup
```

## Step 11: Test restore conflict

```bash
# Restore while resources already exist
velero restore create restore-conflict \
  --from-backup backup-test-ns \
  --wait

# Check errors
velero restore describe restore-conflict --details
```

Shows "AlreadyExists" errors for existing resources!

## Step 12: Test backup hooks

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
  namespace: backup-test
  annotations:
    pre.hook.backup.velero.io/container: db
    pre.hook.backup.velero.io/command: '["/bin/bash", "-c", "echo Flushing DB..."]'
    post.hook.backup.velero.io/container: db
    post.hook.backup.velero.io/command: '["/bin/bash", "-c", "echo DB backup complete"]'
spec:
  containers:
  - name: db
    image: postgres:15
    env:
    - name: POSTGRES_PASSWORD
      value: password
EOF

# Create backup with hooks
velero backup create backup-with-hooks \
  --include-namespaces backup-test \
  --wait

# Check hook execution
velero backup logs backup-with-hooks | grep -i hook
```

## Step 13: Test scheduled backups

```bash
# Create backup schedule
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces backup-test \
  --ttl 168h

# Check schedule
velero schedule get

# Trigger manual run
velero backup create --from-schedule daily-backup --wait
```

## Step 14: Test backup exclusions

```bash
# Backup excluding certain resources
velero backup create exclude-test \
  --include-namespaces backup-test \
  --exclude-resources pods,replicasets \
  --wait

velero backup describe exclude-test
```

Only namespace-level resources backed up!

## Step 15: Check backup storage location

```bash
# Get backup storage location
velero backup-location get

# Check if backup succeeded
velero backup get

# Describe backup details
velero backup describe backup-test-ns --details

# Check for warnings/errors
velero backup logs backup-test-ns | grep -i "error\|warning\|failed"
```

## Key Observations

✅ **Volume snapshots** - require cloud provider integration
✅ **emptyDir** - not backed up (ephemeral storage)
✅ **Restore conflicts** - won't update existing resources
✅ **Hooks** - enable application-consistent backups
✅ **Selective backup** - use labels/namespaces
✅ **TTL** - auto-deletes old backups

## Production Patterns

**Complete backup configuration:**
```bash
velero backup create production-backup \
  --include-namespaces production \
  --include-cluster-resources=true \
  --snapshot-volumes \
  --ttl 720h \
  --storage-location default \
  --volume-snapshot-locations default \
  --labels environment=production
```

**Scheduled backup with retention:**
```bash
velero schedule create nightly \
  --schedule="0 1 * * *" \
  --include-namespaces production,staging \
  --ttl 2160h \
  --include-cluster-resources=true
```

**Disaster recovery procedure:**
```bash
# 1. Install Velero in new cluster
velero install --provider aws --bucket my-backups --secret-file ./credentials

# 2. List available backups
velero backup get

# 3. Restore specific backup
velero restore create dr-restore-$(date +%s) \
  --from-backup production-backup-20240101 \
  --wait

# 4. Verify restoration
kubectl get all -A
velero restore describe dr-restore-* --details
```

**Application-consistent database backup:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres
  annotations:
    # Flush DB before backup
    pre.hook.backup.velero.io/command: |
      ["/bin/bash", "-c", "PGPASSWORD=$POSTGRES_PASSWORD pg_dump -U postgres -d mydb > /backup/dump.sql"]
    pre.hook.backup.velero.io/container: postgres
    # Verify after backup
    post.hook.backup.velero.io/command: '["/bin/bash", "-c", "echo Backup complete"]'
spec:
  containers:
  - name: postgres
    image: postgres:15
```

**Velero with restic for PV backup:**
```bash
# Install with restic
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket velero \
  --secret-file credentials \
  --use-node-agent \
  --use-volume-snapshots=false

# Annotate pods for restic backup
kubectl annotate pod -n myapp mypod \
  backup.velero.io/backup-volumes=data,logs
```

**Monitor backup health:**
```yaml
# Prometheus alert
- alert: VeleroBackupFailed
  expr: velero_backup_failure_total > 0
  for: 15m
  annotations:
    summary: "Velero backup failures detected"

- alert: VeleroBackupTooOld
  expr: time() - velero_backup_last_successful_timestamp > 86400
  annotations:
    summary: "No successful backup in 24 hours"
```

## Cleanup

```bash
velero backup delete --all --confirm
velero schedule delete --all --confirm
kubectl delete namespace backup-test
velero uninstall --force
kubectl delete namespace velero
rm -f velero-v1.12.0-linux-amd64.tar.gz
```

---
Next: Day 50 - Node Failures and Pod Eviction Storms
