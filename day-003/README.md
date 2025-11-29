## Day 3: OOMKilled - When Memory Limits Bite

## THE IDEA
Set a 128MB memory limit and watch the Linux OOM killer terminate our container
when it tries to allocate 256MB. Spoiler: Kubernetes doesn’t warn you first.

## THE SETUP
Deploy a pod that intentionally leaks memory. We’ll see the container get
OOMKilled, then explore the difference between requests and limits.

## WHAT I LEARNED
- OOMKill happens instantly when you exceed limits (no grace period)
- “requests” guarantee resources, “limits” cap maximum usage
- QoS classes matter: Guaranteed > Burstable > BestEffort
- Exit code 137 = killed by OOM (128 + SIGKILL 9)

## WHY IT MATTERS
Misconfigured limits cause:
- Random pod evictions under load
- Slow performance (CPU throttling)
- Cluster instability from cascading failures

TRY IT:
🧪 Interactive Lab: [https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-003](https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-003)

---

## Killercoda Lab Instructions

**Step 1: Start a pod that is a memory hog:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: memory-hog
spec:
  containers:
  - name: stress
    image: polinux/stress
    resources:
      requests:
        memory: "64Mi"
      limits:
        memory: "128Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "256M", "--vm-hang", "1"]
EOF

```

**Step 2: Watch the pod get OOMKilled**

```bash
kubectl get pods memory-hog -w

```

You’ll see: Pending → ContainerCreating → Running → OOMKilled → CrashLoopBackOff

The pod tries to allocate 256MB but is limited to 128MB.

**Step 3: Check the termination reason**

```bash
kubectl describe pod memory-hog | grep -A 5 "Last State"

```

Look for:

- Reason: OOMKilled
- Exit Code: 137 (128 + 9 for SIGKILL)

**Step 4: View container resource usage before termination**

```bash
kubectl top pod memory-hog

```

(May show “error” if pod is restarting)

**Step 5: Create a pod with matching requests and limits (Guaranteed QoS)**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: memory-guaranteed
spec:
  containers:
  - name: stress
    image: polinux/stress
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "100m"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "100M", "--vm-hang", "1"]
EOF

```

**Check the QoS class:**

```bash
kubectl get pod memory-guaranteed -o jsonpath='{.status.qosClass}'

```

Output: `Guaranteed`

**Step 6: Create a pod with no limits (BestEffort QoS)**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: memory-besteffort
spec:
  containers:
  - name: stress
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "100M", "--vm-hang", "1"]
EOF

```

**Check QoS:**

```bash
kubectl get pod memory-besteffort -o jsonpath='{.status.qosClass}'

```

Output: `BestEffort` (first to be evicted under node pressure!)

**Step 7: Simulate node memory pressure**

```bash
kubectl describe node | grep -A 5 "Allocated resources"

```

Under memory pressure, K8s evicts pods in this order:

1. BestEffort pods (no requests/limits)
2. Burstable pods exceeding requests
3. Guaranteed pods (last resort)

**Key Observations**

✅ **OOMKill is instant** - no warnings, container dies mid-operation
✅ **Exit code 137** - reliable indicator of OOM issues
✅ **QoS classes** - determine eviction priority under node pressure
✅ **Guaranteed QoS** - requests == limits for all containers

**Production Tips**

**Set both requests and limits for predictable behavior:**

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "200m"
  limits:
    memory: "512Mi"
    cpu: "500m"

```

**For burstable workloads (web servers):**

- Set low requests (guaranteed minimum)
- Set high limits (burst capacity)

**For consistent workloads (databases):**

- Set requests == limits (Guaranteed QoS)