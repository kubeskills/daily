## Day 2: StatefulSet Pod Ordering - Break the Sequence

### THE IDEA:
StatefulSets create pods in strict order: 0, then 1, then 2. What happens when
we force-delete pod-1 while pod-2 is starting? Does K8s panic?

### THE SETUP:
Create a 3-replica StatefulSet with slow startup (sleep 30s). While pod-2 is
pending, we’ll force-delete pod-1 and watch the reconciliation dance.

### WHAT I LEARNED:
- K8s immediately recreates pod-1 but keeps pod-2 in Pending
- Pod-2 only starts after pod-1 is Running again
- Force deletion (–grace-period=0 –force) leaves zombie pods in etcd
- The .spec.podManagementPolicy field controls this (OrderedReady vs Parallel)

### WHY IT MATTERS:

Understanding ordering is critical for:
- Databases requiring leader election
- Distributed systems with consensus protocols
- Any app where startup sequence matters

### TRY IT:
🧪 Interactive Lab: [https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-002](https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-002)


---

## Killercoda Lab Instructions

### Step 1: Create a StatefulSet with slow-starting pods

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 3
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
        image: nginx:1.25
        command: ['sh', '-c', 'echo "Pod starting..."; sleep 30; nginx -g "daemon off;"']
        ports:
        - containerPort: 80
EOF

```

### **Step 2: Watch the ordered creation**

```bash
kubectl get pods -l app=nginx -w

```

Notice pods appear sequentially: web-0, then web-1, then web-2.
Each waits for the previous to be Running before starting.

Keep this terminal open. Open a new terminal for the next step.

### **Step 3: Force-delete pod-1 while pod-2 is starting**

Wait until web-1 is Running and web-2 shows “ContainerCreating”, then:

```bash
kubectl delete pod web-1 --grace-period=0 --force

```

**Observe:** web-2 remains stuck! It won’t become Running until web-1 is healthy again.

### **Step 4: Check the StatefulSet controller events**

```bash
kubectl describe statefulset web | tail -20

```

Look for messages about waiting for predecessors.

### **Step 5: Verify pod-2 eventually starts**

Once web-1 is Running again:

```bash
kubectl get pods -l app=nginx

```

All three pods should now be Running.

### **Step 6: Test Parallel pod management**

```bash
kubectl delete statefulset web

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 3
  podManagementPolicy: Parallel  # Changed from OrderedReady
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
        image: nginx:1.25
        ports:
        - containerPort: 80
EOF

```

**Watch the difference:**

```bash
kubectl get pods -l app=nginx -w

```

All three pods start simultaneously!

Key Observations

✅ **OrderedReady** (default): Strict ordering, web-1 must be Running before web-2 starts
✅ **Parallel**: All pods start at once, no ordering guarantees
✅ Force deletion can leave pods in Unknown state, breaking ordering assumptions

Production Tip

Use OrderedReady for:

- Databases (PostgreSQL, MySQL with replication)
- Consensus systems (etcd, ZooKeeper, Consul)

Use Parallel for:

- Stateless apps incorrectly using StatefulSets
- Faster scaling when order doesn’t matter

### Cleanup (Optional)

```bash
kubectl delete statefulset web

```

---

Next: Day 3 - OOM Killer vs Resource Limits