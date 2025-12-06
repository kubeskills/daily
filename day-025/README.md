## Day 25: etcd Performance - The Cluster Heartbeat

## THE IDEA:
Simulate etcd performance degradation and watch the entire cluster slow down.
API server requests timeout, controllers lag, and deployments hang.

## THE SETUP:
Check etcd health metrics, simulate slow disk I/O, and observe how etcd latency
cascades through every cluster operation.

## WHAT I LEARNED: 
- etcd stores ALL cluster state (pods, services, secrets, etc.)
- Slow etcd = slow API server = cluster-wide impact
- Raft consensus requires quorum (N/2 + 1 nodes)
- Disk latency is etcd's biggest enemy

## WHY IT MATTERS:
etcd issues cause:
- API server timeouts and failures
- Controllers unable to reconcile state
- Complete cluster unresponsiveness

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-025

Tomorrow: kubelet eviction thresholds and resource pressure.

---

# Killercoda Lab Instructions

## Step 1: Check etcd pods

```bash
kubectl get pods -n kube-system -l component=etcd

```

**Check etcd endpoints:**

```bash
kubectl get endpoints -n kube-system | grep etcd

```

## Step 2: Access etcd health endpoint

```bash
# Get etcd pod name
ETCD_POD=$(kubectl get pods -n kube-system -l component=etcd -o jsonpath='{.items[0].metadata.name}')

# Check health
kubectl exec -n kube-system $ETCD_POD -- sh -c 'ETCDCTL_API=3 etcdctl \\
  --endpoints=https://127.0.0.1:2379 \\
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \\
  --cert=/etc/kubernetes/pki/etcd/server.crt \\
  --key=/etc/kubernetes/pki/etcd/server.key \\
  endpoint health'

```

Should show: "healthy"

## Step 3: Check etcd metrics

```bash
kubectl exec -n kube-system $ETCD_POD -- sh -c 'ETCDCTL_API=3 etcdctl \\
  --endpoints=https://127.0.0.1:2379 \\
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \\
  --cert=/etc/kubernetes/pki/etcd/server.crt \\
  --key=/etc/kubernetes/pki/etcd/server.key \\
  endpoint status --write-out=table'

```

Shows:

- Leader election status
- Database size
- Raft index

## Step 4: Check etcd database size

```bash
kubectl exec -n kube-system $ETCD_POD -- sh -c 'du -sh /var/lib/etcd'

```

**Check for large database:**

```bash
kubectl exec -n kube-system $ETCD_POD -- sh -c 'ETCDCTL_API=3 etcdctl \\
  --endpoints=https://127.0.0.1:2379 \\
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \\
  --cert=/etc/kubernetes/pki/etcd/server.crt \\
  --key=/etc/kubernetes/pki/etcd/server.key \\
  endpoint status' | awk '{print $5}'

```

## Step 5: List all keys in etcd

```bash
kubectl exec -n kube-system $ETCD_POD -- sh -c 'ETCDCTL_API=3 etcdctl \\
  --endpoints=https://127.0.0.1:2379 \\
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \\
  --cert=/etc/kubernetes/pki/etcd/server.crt \\
  --key=/etc/kubernetes/pki/etcd/server.key \\
  get / --prefix --keys-only | head -20'

```

Everything in Kubernetes is stored here!

## Step 6: Count resources by type

```bash
kubectl exec -n kube-system $ETCD_POD -- sh -c 'ETCDCTL_API=3 etcdctl \\
  --endpoints=https://127.0.0.1:2379 \\
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \\
  --cert=/etc/kubernetes/pki/etcd/server.crt \\
  --key=/etc/kubernetes/pki/etcd/server.key \\
  get / --prefix --keys-only' | grep -o '/[^/]*/[^/]*/' | sort | uniq -c | sort -rn | head -10

```

Shows which resource types consume most keys.

## Step 7: Measure API server latency

```bash
# Create resources rapidly
time for i in $(seq 1 100); do
  kubectl create configmap test-$i --from-literal=key=value --dry-run=client -o yaml | kubectl apply -f - > /dev/null
done

```

Baseline latency measurement.

## Step 8: Simulate etcd load (create many resources)

```bash
# Create 1000 secrets (heavy on etcd)
for i in $(seq 1 100); do
  kubectl create secret generic load-secret-$i \\
    --from-literal=data=$(head -c 1024 /dev/urandom | base64) &
done
wait

# Check etcd size again
kubectl exec -n kube-system $ETCD_POD -- sh -c 'du -sh /var/lib/etcd'

```

## Step 9: Test API server responsiveness

```bash
# Measure kubectl response time
time kubectl get nodes
time kubectl get pods -A
time kubectl get all -A > /dev/null

```

## Step 10: Check etcd compaction status

```bash
kubectl exec -n kube-system $ETCD_POD -- sh -c 'ETCDCTL_API=3 etcdctl \\
  --endpoints=https://127.0.0.1:2379 \\
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \\
  --cert=/etc/kubernetes/pki/etcd/server.crt \\
  --key=/etc/kubernetes/pki/etcd/server.key \\
  endpoint status -w table'

```

Look for revision numbers.

## Step 11: Manually compact etcd (free space)

```bash
# Get current revision
REVISION=$(kubectl exec -n kube-system $ETCD_POD -- sh -c 'ETCDCTL_API=3 etcdctl \\
  --endpoints=https://127.0.0.1:2379 \\
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \\
  --cert=/etc/kubernetes/pki/etcd/server.crt \\
  --key=/etc/kubernetes/pki/etcd/server.key \\
  endpoint status' | awk '{print $4}')

echo "Current revision: $REVISION"

# Compact (keeps last 1000 revisions)
kubectl exec -n kube-system $ETCD_POD -- sh -c "ETCDCTL_API=3 etcdctl \\
  --endpoints=https://127.0.0.1:2379 \\
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \\
  --cert=/etc/kubernetes/pki/etcd/server.crt \\
  --key=/etc/kubernetes/pki/etcd/server.key \\
  compact $((REVISION - 1000))"

```

## Step 12: Defragment etcd

```bash
kubectl exec -n kube-system $ETCD_POD -- sh -c 'ETCDCTL_API=3 etcdctl \\
  --endpoints=https://127.0.0.1:2379 \\
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \\
  --cert=/etc/kubernetes/pki/etcd/server.crt \\
  --key=/etc/kubernetes/pki/etcd/server.key \\
  defrag'

```

**Check size after defrag:**

```bash
kubectl exec -n kube-system $ETCD_POD -- sh -c 'du -sh /var/lib/etcd'

```

Should be smaller!

## Step 13: Backup etcd

```bash
kubectl exec -n kube-system $ETCD_POD -- sh -c 'ETCDCTL_API=3 etcdctl \\
  --endpoints=https://127.0.0.1:2379 \\
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \\
  --cert=/etc/kubernetes/pki/etcd/server.crt \\
  --key=/etc/kubernetes/pki/etcd/server.key \\
  snapshot save /tmp/etcd-backup.db'

# Copy backup out
kubectl cp -n kube-system $ETCD_POD:/tmp/etcd-backup.db ./etcd-backup.db

echo "Backup saved to ./etcd-backup.db"
ls -lh ./etcd-backup.db

```

## Step 14: Check etcd alarms

```bash
kubectl exec -n kube-system $ETCD_POD -- sh -c 'ETCDCTL_API=3 etcdctl \\
  --endpoints=https://127.0.0.1:2379 \\
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \\
  --cert=/etc/kubernetes/pki/etcd/server.crt \\
  --key=/etc/kubernetes/pki/etcd/server.key \\
  alarm list'

```

Common alarm: `NOSPACE` (database too large)

## Step 15: Monitor etcd metrics via API server

```bash
# API server exposes etcd metrics
kubectl get --raw /metrics | grep etcd | head -20

```

Key metrics:

- `etcd_request_duration_seconds`
- `etcd_disk_backend_commit_duration_seconds`
- `etcd_disk_wal_fsync_duration_seconds`

## Key Observations

✅ **etcd is single source of truth** - all cluster state stored here
✅ **Disk latency matters** - SSD recommended, fsync < 10ms critical
✅ **Database size** - compact regularly, limit to 8GB
✅ **Backup frequently** - use snapshot for disaster recovery
✅ **Quorum required** - N/2 + 1 nodes must be healthy

## Production Patterns

**Automated etcd backup:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          containers:
          - name: backup
            image: k8s.gcr.io/etcd:3.5.9-0
            command:
            - /bin/sh
            - -c
            - |
              ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db
          volumeMounts:
          - name: etcd-certs
            mountPath: /etc/kubernetes/pki/etcd
          - name: backup
            mountPath: /backup
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
          - name: backup
            hostPath:
              path: /var/backups/etcd
          restartPolicy: OnFailure

```

**etcd compaction automation:**

```bash
# Cron job to compact daily
0 3 * * * etcdctl compact $(etcdctl endpoint status --write-out="json" | jq -r '.[0].Status.header.revision - 1000')

```

**Monitor etcd health:**

```bash
# Alert on etcd latency > 100ms
etcd_disk_backend_commit_duration_seconds_bucket{le="0.1"} < 0.99

```

## Cleanup

```bash
# Clean up test resources
kubectl delete configmap $(kubectl get cm -o name | grep test-) 2>/dev/null
kubectl delete secret $(kubectl get secret -o name | grep load-secret-) 2>/dev/null
rm -f ./etcd-backup.db

```

---

Next: Day 26 - Kubelet Eviction Thresholds