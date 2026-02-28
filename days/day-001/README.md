# Welcome to Day 1! 🚀

## THE IDEA:
Test how Kubernetes handles failing containers with different restart policies.
What happens when your app crashes? Does K8s retry forever, give up, or
something in between?

## THE SETUP:
We'll create three pods with intentionally failing containers, each using a
different restart policy: Always, OnFailure, and Never.

## WHAT I LEARNED:
- "Always" restarts even on exit code 0 (successful completion)
- "OnFailure" only restarts on non-zero exits
- "Never" leaves failed pods around (useful for debugging)
- Backoff delay increases exponentially (10s, 20s, 40s... up to 5min)

## WHY IT MATTERS:
Choosing the wrong restart policy can cause:
- Jobs that never complete (Always on batch workloads)
- Lost debugging info (Always on one-shot scripts)
- Resource exhaustion from restart loops

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-001

---

## Killercoda Lab Instructions

### Scenario Setup
We'll test three restart policies by creating pods that exit with different codes.

### Step 1: Create a pod with "Always" restart policy

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: restart-always
spec:
  restartPolicy: Always
  containers:
  - name: fail-container
    image: busybox
    command: ['sh', '-c', 'echo "Running..."; sleep 5; exit 0']
EOF

```

**Watch the behavior:**

```bash
kubectl get pods restart-always -w

```

Notice: Even though the container exits successfully (code 0), K8s restarts it.
Press Ctrl+C after ~30 seconds.

### Step 2: Create a pod with “OnFailure” restart policy

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: restart-onfailure
spec:
  restartPolicy: OnFailure
  containers:
  - name: fail-container
    image: busybox
    command: ['sh', '-c', 'echo "Running..."; sleep 5; exit 1']
EOF

```

**Watch the restart behavior:**

```bash
kubectl get pods restart-onfailure -w

```

This one restarts because exit code 1 = failure. Notice the restart count increasing.
Press Ctrl+C after ~30 seconds.

### Step 3: Create a pod with “Never” restart policy

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: restart-never
spec:
  restartPolicy: Never
  containers:
  - name: fail-container
    image: busybox
    command: ['sh', '-c', 'echo "Running..."; sleep 5; exit 1']
EOF

```

**Check the status:**

```bash
kubectl get pods restart-never

```

Pod stays in “Error” state. No restart. Perfect for debugging!

### Step 4: Observe the backoff delay

```bash
kubectl describe pod restart-onfailure | grep -A 10 Events

```

Look for the “Back-off restarting failed container” messages.
The delay increases: 10s → 20s → 40s → 80s → 160s → 300s (max).


### Step 5: Check logs from previous container run

```bash
kubectl logs restart-onfailure --previous

```

The `--previous` flag shows logs from the last terminated container.

**Key Observations**

✅ **Always**: Restarts on any exit (even success). Use for long-running services.
✅ **OnFailure**: Restarts only on non-zero exit codes. Use for Jobs/batch workloads.
✅ **Never**: No restarts. Use for debugging or one-shot init tasks.


### Cleanup (Optional)

```bash
kubectl delete pod restart-always restart-onfailure restart-never

```

**Production Tip**

For Jobs, use `restartPolicy: OnFailure` or `Never`. Using `Always` in a Job
will cause it to never complete!

---

Next: [Day 2 - Breaking StatefulSet Ordering](../day-002/README.md)
