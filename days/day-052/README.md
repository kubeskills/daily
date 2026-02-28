## Day 52: etcd Corruption - Quorum Loss Disaster

### Email Copy

**Subject:** Day 52: etcd Corruption - When the Database Dies

THE IDEA:
Simulate etcd member failures and quorum loss. Watch the cluster freeze completely
when etcd can't reach consensus, unable to read or write any state.

THE SETUP:
Check etcd cluster health, simulate member failures, trigger quorum loss, and
test recovery procedures. Learn what breaks when the database fails.

WHAT I LEARNED:
- etcd requires N/2 + 1 members for quorum (3 members = need 2)
- Losing quorum = cluster completely frozen (no reads or writes)
- Disk corruption can take down individual members
- Snapshot restore requires cluster downtime
- Regular backups are the only disaster recovery

WHY IT MATTERS:
etcd failures cause:
- Complete cluster outage (can't create/update/delete anything)
- Data loss if no recent backups
- Complex recovery requiring manual intervention

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-052

Tomorrow: Network partition (split-brain) scenarios.

---
Chad


---

## Killercoda Lab Instructions




## Step 1: Check etcd cluster health

```bash
# Get etcd pod
ETCD_POD=$(kubectl get pods -n kube-system -l component=etcd -o jsonpath='{.items[0].metadata.name}')

# Check etcd health
kubectl exec -n kube-system $ETCD_POD -- sh -c \
  'ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health'
```

## Step 2: Check etcd member list

```bash
kubectl exec -n kube-system $ETCD_POD -- sh -c \
  'ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list -w table'
```

## Step 3: Check etcd database size

```bash
kubectl exec -n kube-system $ETCD_POD -- sh -c \
  'ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status -w table'
```

## Step 4: Create etcd snapshot (backup)

```bash
kubectl exec -n kube-system $ETCD_POD -- sh -c \
  'ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /tmp/etcd-backup.db'

# Copy snapshot out
kubectl cp -n kube-system $ETCD_POD:/tmp/etcd-backup.db ./etcd-backup.db

echo "Backup saved locally"
ls -lh etcd-backup.db
```

## Step 5: Test etcd performance

```bash
# Check etcd metrics
kubectl exec -n kube-system $ETCD_POD -- sh -c \
  'ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  check perf'
```

## Step 6: Simulate high etcd load

```bash
# Create many resources rapidly
for i in $(seq 1 100); do
  kubectl create configmap load-test-$i --from-literal=data=$(date) &
done
wait

# Check etcd size growth
kubectl exec -n kube-system $ETCD_POD -- sh -c \
  'du -sh /var/lib/etcd'
```

## Step 7: Test etcd compaction

```bash
# Get current revision
REVISION=$(kubectl exec -n kube-system $ETCD_POD -- sh -c \
  'ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status' | awk '{print $4}')

echo "Current revision: $REVISION"

# Compact old revisions
kubectl exec -n kube-system $ETCD_POD -- sh -c \
  "ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  compact $REVISION"
```

## Step 8: Test etcd defragmentation

```bash
# Check size before defrag
kubectl exec -n kube-system $ETCD_POD -- sh -c 'du -sh /var/lib/etcd'

# Defragment
kubectl exec -n kube-system $ETCD_POD -- sh -c \
  'ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  defrag'

# Check size after
kubectl exec -n kube-system $ETCD_POD -- sh -c 'du -sh /var/lib/etcd'
```

## Step 9: Check etcd alarms

```bash
kubectl exec -n kube-system $ETCD_POD -- sh -c \
  'ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  alarm list'
```

Common alarm: NOSPACE (database quota exceeded)

## Step 10: Test etcd metrics

```bash
# Get etcd metrics
kubectl exec -n kube-system $ETCD_POD -- sh -c \
  'curl -k --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  https://127.0.0.1:2379/metrics' | grep -E "etcd_server|etcd_disk" | head -20
```

## Step 11: Simulate quorum loss (conceptual)

```bash
# In 3-member cluster, losing 2 members = quorum loss
# Cluster becomes completely unavailable

# Conceptual example:
# If you have access to multiple etcd members:
# Stop 2 out of 3 etcd processes
# All API calls will hang/fail

echo "Quorum loss simulation:"
echo "3-member cluster needs 2 healthy members (N/2 + 1)"
echo "5-member cluster needs 3 healthy members"
echo "Losing quorum = cluster frozen (no reads/writes)"
```

## Step 12: Test snapshot restore (dry run)

```bash
# This is destructive! Only for learning
echo "Snapshot restore procedure (DO NOT RUN in production):"
cat << 'EOF'
# 1. Stop API server
systemctl stop kube-apiserver

# 2. Restore snapshot
ETCDCTL_API=3 etcdctl snapshot restore etcd-backup.db \
  --data-dir=/var/lib/etcd-restored \
  --name=etcd-0 \
  --initial-cluster=etcd-0=https://127.0.0.1:2380 \
  --initial-advertise-peer-urls=https://127.0.0.1:2380

# 3. Move old data
mv /var/lib/etcd /var/lib/etcd-old
mv /var/lib/etcd-restored /var/lib/etcd

# 4. Start etcd
systemctl start etcd

# 5. Start API server
systemctl start kube-apiserver

# 6. Verify
kubectl get nodes
EOF
```

## Step 13: Check etcd consistency

```bash
# Get hash of data
kubectl exec -n kube-system $ETCD_POD -- sh -c \
  'ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint hash'
```

## Step 14: Monitor etcd write performance

```bash
# Watch etcd write latency
watch -n 2 'kubectl exec -n kube-system $ETCD_POD -- sh -c \
  "curl -k --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  https://127.0.0.1:2379/metrics" | grep etcd_disk_backend_commit_duration_seconds_bucket | tail -5'
```

## Step 15: Document disaster recovery procedure

```bash
cat > /tmp/etcd-dr-plan.md << 'EOF'
# etcd Disaster Recovery Plan

## Backup Procedure (Daily)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

## Recovery from Snapshot
1. Stop kube-apiserver
2. Restore snapshot to new data directory
3. Update etcd to use new data directory
4. Start etcd
5. Start kube-apiserver
6. Verify cluster state

## Recovery from Quorum Loss
- If quorum lost but data intact: restore from healthy member
- If data corrupted: restore from latest snapshot (some data loss)
- RTO: 15-30 minutes
- RPO: Last successful backup (24 hours max)

## Monitoring
- Alert on etcd_server_has_leader == 0
- Alert on etcd_disk_wal_fsync_duration_seconds > 100ms
- Alert on etcd database size > 7GB
EOF

cat /tmp/etcd-dr-plan.md
```

## Key Observations

✅ **Quorum required** - N/2 + 1 members must be healthy
✅ **Losing quorum** - cluster completely frozen
✅ **Backups critical** - only way to recover from corruption
✅ **Compaction needed** - prevents database growth
✅ **Disk performance** - critical for etcd health
✅ **Snapshot restore** - requires cluster downtime

## Production Patterns

**Automated backup:**
```bash
#!/bin/bash
# etcd-backup.sh

BACKUP_DIR="/backup/etcd"
RETENTION_DAYS=7

# Create backup
ETCDCTL_API=3 etcdctl snapshot save \
  $BACKUP_DIR/etcd-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status \
  $BACKUP_DIR/etcd-$(date +%Y%m%d-*)*.db

# Delete old backups
find $BACKUP_DIR -name "etcd-*.db" -mtime +$RETENTION_DAYS -delete

# Upload to S3
aws s3 sync $BACKUP_DIR s3://my-etcd-backups/
```

**Monitoring alerts:**
```yaml
# Prometheus alerts
- alert: etcdNoLeader
  expr: etcd_server_has_leader == 0
  for: 1m
  annotations:
    summary: "etcd has no leader"

- alert: etcdHighFsyncDuration
  expr: histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])) > 0.5
  for: 10m
  annotations:
    summary: "etcd fsync duration too high"

- alert: etcdDatabaseSizeLimit
  expr: etcd_mvcc_db_total_size_in_bytes > 7516192768  # 7GB
  annotations:
    summary: "etcd database approaching size limit"
```

**Health check script:**
```bash
#!/bin/bash
# etcd-health-check.sh

ENDPOINTS="https://127.0.0.1:2379"

# Check member health
HEALTHY=$(ETCDCTL_API=3 etcdctl \
  --endpoints=$ENDPOINTS \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health | grep -c "is healthy")

if [ $HEALTHY -eq 0 ]; then
  echo "CRITICAL: etcd unhealthy"
  exit 2
fi

# Check alarm status
ALARMS=$(ETCDCTL_API=3 etcdctl \
  --endpoints=$ENDPOINTS \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  alarm list)

if [ -n "$ALARMS" ]; then
  echo "WARNING: etcd alarms present: $ALARMS"
  exit 1
fi

echo "OK: etcd healthy"
exit 0
```

## Cleanup

```bash
kubectl delete configmap $(kubectl get cm -o name | grep load-test-) 2>/dev/null
rm -f etcd-backup.db /tmp/etcd-dr-plan.md
```

---
Next: Day 53 - Network Partition (Split-Brain) Scenarios
