## Day 53: Network Partitions - Split-Brain Nightmare

### Email Copy

**Subject:** Day 53: Network Partitions - When the Cluster Splits in Two

THE IDEA:
Simulate network partitions between nodes and watch split-brain scenarios unfold.
Discover duplicate StatefulSet pods, quorum issues, and data inconsistency.

THE SETUP:
Use NetworkPolicy to isolate pods, test cross-zone failures, and observe what
happens when nodes can't communicate but all stay "healthy".

WHAT I LEARNED:
- Network partition != node failure (nodes stay Ready)
- StatefulSets can create duplicate pods in different partitions
- Quorum-based systems (etcd, databases) protect against split-brain
- DNS resolution continues (but endpoints unreachable)
- Some apps handle partitions gracefully, others corrupt data

WHY IT MATTERS:
Network partitions cause:
- Data corruption from concurrent writes
- Duplicate workloads wasting resources
- Subtle bugs that only appear during rare failures

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-053

Tomorrow: Certificate rotation failures (Week 8 finale!).

---
Chad
Kube Daily - 53 days of controlled chaos = expertise


---

## Killercoda Lab Instructions




## Step 1: Deploy distributed application

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: split-test-svc
spec:
  clusterIP: None
  selector:
    app: split-test
  ports:
  - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: split-test
spec:
  serviceName: split-test-svc
  replicas: 3
  selector:
    matchLabels:
      app: split-test
  template:
    metadata:
      labels:
        app: split-test
    spec:
      containers:
      - name: app
        image: nginx
EOF
```

<!-- Content for this day was not fully provided. Please add the remaining steps. -->
