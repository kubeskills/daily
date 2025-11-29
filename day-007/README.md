## Day 7: Init Containers - Blocking Your Main App

### THE IDEA:
Create an init container that waits for a service that doesn’t exist. Main
container never starts. Then fix it with proper timeout handling and see the
difference between Init vs Sidecar containers.

### THE SETUP:
Deploy a pod with an init container that runs `nslookup nonexistent-service` in
a loop. Watch the pod get stuck in Init state indefinitely. Explore restart policies.

### WHAT I LEARNED:
- Init containers run sequentially, blocking main container
- Failed init containers respect restartPolicy (OnFailure vs Never)
- Sidecars (K8s 1.28+) run alongside main container
- No timeout mechanism - init containers can block forever

### WHY IT MATTERS:
Common init container anti-patterns:
- Database connection checks without timeouts
- External API dependency checks (services down at deploy time)
- Complex logic that should be in the main app

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-007

---

## Killercoda Lab Instructions

### Step 1: Deploy a pod that never unblocks

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: blocked-pod
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nslookup nonexistent-db; do echo waiting; sleep 2; done']
  containers:
  - name: app
    image: nginx
    ports:
    - containerPort: 80
EOF

```

**Watch it get stuck:**

```bash
kubectl get pods blocked-pod -w

```

Status: Init:0/1 (stuck forever!)

### Step 2: Check init container logs

```bash
kubectl logs blocked-pod -c wait-for-db

```

You’ll see endless “waiting” messages. The main nginx container never starts.

### Step 3: Describe the pod

```bash
kubectl describe pod blocked-pod

```

Look at the “Init Containers” section - shows the container is running but never completes.

### Step 4: Fix with a timeout

```bash
kubectl delete pod blocked-pod

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: fixed-pod
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c']
    args:
    - |
      for i in $(seq 1 10); do
        if nslookup nonexistent-db; then
          echo "Service found!"
          exit 0
        fi
        echo "Attempt $i/10 failed, retrying..."
        sleep 3
      done
      echo "Service not found after 10 attempts, giving up"
      exit 1
  containers:
  - name: app
    image: nginx
    ports:
    - containerPort: 80
EOF

```

**Watch the behavior:**

```bash
kubectl get pods fixed-pod -w

```

Init container fails after 30s, pod enters CrashLoopBackOff (with restartPolicy: Always, it keeps retrying).

### **Step 5: Change restart policy to Never**

```bash
kubectl delete pod fixed-pod

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-restart-pod
spec:
  restartPolicy: Never
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c']
    args:
    - |
      for i in $(seq 1 5); do
        if nslookup nonexistent-db; then exit 0; fi
        echo "Attempt $i/5 failed"
        sleep 2
      done
      exit 1
  containers:
  - name: app
    image: nginx
EOF

```

**Check final state:**

```bash
kubectl get pods no-restart-pod

```

Status: Init:Error (stays in this state, no retry)

### **Step 6: Multiple init containers (sequential execution)**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multi-init
spec:
  initContainers:
  - name: init-1
    image: busybox
    command: ['sh', '-c', 'echo "Init 1 starting"; sleep 5; echo "Init 1 done"']
  - name: init-2
    image: busybox
    command: ['sh', '-c', 'echo "Init 2 starting"; sleep 5; echo "Init 2 done"']
  - name: init-3
    image: busybox
    command: ['sh', '-c', 'echo "Init 3 starting"; sleep 5; echo "Init 3 done"']
  containers:
  - name: app
    image: nginx
EOF

```

**Watch sequential execution:**

```bash
kubectl get pods multi-init -w

```

You’ll see: Init:0/3 → Init:1/3 → Init:2/3 → Init:3/3 → Running

Each init container runs to completion before the next starts.

**View logs from all init containers:**

```bash
kubectl logs multi-init -c init-1
kubectl logs multi-init -c init-2
kubectl logs multi-init -c init-3

```

### **Step 7: Sidecar containers (K8s 1.28+)**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
spec:
  initContainers:
  - name: sidecar-logger
    image: busybox
    restartPolicy: Always  # Key difference - marks it as sidecar
    command: ['sh', '-c', 'while true; do echo "$(date) - Sidecar running"; sleep 5; done']
  containers:
  - name: app
    image: nginx
EOF

```

**Check both containers running:**

```bash
kubectl get pods sidecar-demo -o jsonpath='{.status.containerStatuses[*].name}'
kubectl logs sidecar-demo -c sidecar-logger -f

```

Sidecar continues running alongside main container!

**Key Observations**

✅ **Init containers run sequentially** - each must complete before next starts
✅ **Block main container** - app doesn’t start until all init containers succeed
✅ **Respect restartPolicy** - but restartPolicy applies to the whole pod
✅ **No built-in timeouts** - you must implement retry logic with limits
✅ **Sidecars (1.28+)** - restartPolicy: Always in initContainers makes them sidecars

**Production Patterns**

**DB connection check with timeout:**

```yaml
initContainers:
- name: wait-for-postgres
  image: postgres:15
  command:
  - sh
  - -c
  - |
    for i in $(seq 1 30); do
      if pg_isready -h postgres-svc -p 5432; then
        echo "Postgres ready!"
        exit 0
      fi
      echo "Waiting for Postgres... ($i/30)"
      sleep 2
    done
    exit 1

```

**Download config from remote:**

```yaml
initContainers:
- name: fetch-config
  image: curlimages/curl
  command:
  - sh
  - -c
  - curl -o /config/app.conf <https://config-server/app.conf>
  volumeMounts:
  - name: config
    mountPath: /config

```

**Database migration before app starts:**

```yaml
initContainers:
- name: db-migrate
  image: migrate/migrate
  command:
  - migrate
  - -path=/migrations
  - -database=postgres://...
  - up

```

### Cleanup

```bash
kubectl delete pod blocked-pod fixed-pod no-restart-pod multi-init sidecar-demo

```

---

🎉 Week 1 Complete! You’ve tested 7 failure modes in 7 days.

Next Week Preview:
- Day 8: PVC binding failures and stuck provisioning
- Day 9: ConfigMap hot-reload races
- Day 10: Secret rotation without restarts
- Day 11: Service DNS propagation delays
- Day 12: Horizontal Pod Autoscaler thrashing
- Day 13: PodDisruptionBudget blocking node drains
- Day 14: Resource quota limits and scheduling failures



Next: Day 8 - PVC Binding Failures