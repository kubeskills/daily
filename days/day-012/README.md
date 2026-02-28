# Day 12: HPA Thrashing - The Scale Storm

## THE IDEA:
Configure an HPA with aggressive thresholds and watch it scale up and down
repeatedly, destabilizing your application. Then tune it to behave sensibly.

## THE SETUP:
Deploy an app with HPA targeting 50% CPU. Generate load, watch scale-up, remove
load, watch scale-down, repeat. The constant churn breaks things.

## WHAT I LEARNED:
- Default scale-down stabilization is 300 seconds (5 minutes)
- Scale-up happens faster than scale-down (asymmetric by design)
- Container resource requests directly affect HPA percentages
- Multiple metrics can create conflicting scaling decisions

### WHY IT MATTERS:
HPA thrashing causes:
- Connection storms as pods constantly join/leave load balancers
- Increased cloud costs from rapid provisioning
- Application instability during scale transitions

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-012


---

## Killercoda Lab Instructions

### Prerequisites

Verify metrics-server is running:

```bash
kubectl top nodes

```

If not installed:

```bash
kubectl apply -f <https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml>
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

```

Wait 60 seconds for metrics to populate.

### Step 1: Deploy a CPU-intensive app

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cpu-app
  template:
    metadata:
      labels:
        app: cpu-app
    spec:
      containers:
      - name: app
        image: vish/stress
        resources:
          requests:
            cpu: 100m
          limits:
            cpu: 200m
        args:
        - -cpus
        - "0"  # Idle initially
---
apiVersion: v1
kind: Service
metadata:
  name: cpu-app
spec:
  selector:
    app: cpu-app
  ports:
  - port: 80
    targetPort: 8080
EOF

```

### Step 2: Create an aggressive HPA

```bash
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cpu-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpu-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 30  # Very aggressive!
EOF

```

**Watch HPA status:**

```bash
kubectl get hpa cpu-app-hpa -w

```

### Step 3: Generate load to trigger scale-up

```bash
# Terminal 2: Generate CPU load
kubectl run load-generator --rm -it --restart=Never --image=busybox -- sh -c '
while true; do
  wget -q -O- <http://cpu-app> > /dev/null 2>&1 &
done
'

```

Alternative if the app doesn't serve HTTP - scale the deployment to trigger resource pressure:

```bash
kubectl set env deployment/cpu-app STRESS_CPU=1
kubectl patch deployment cpu-app -p '{"spec":{"template":{"spec":{"containers":[{"name":"app","args":["-cpus","1"]}]}}}}'

```

**Watch scaling:**

```bash
kubectl get hpa,pods -l app=cpu-app -w

```

### Step 4: Remove load and watch thrashing

Stop the load generator (Ctrl+C), then:

```bash
kubectl patch deployment cpu-app -p '{"spec":{"template":{"spec":{"containers":[{"name":"app","args":["-cpus","0"]}]}}}}'

```

**Keep watching:**

- Scale-up happened quickly
- Scale-down waits for stabilization window (default 300s)
- Reapply load before scale-down completes = thrashing!

### Step 5: Check HPA events

```bash
kubectl describe hpa cpu-app-hpa | tail -20

```

Look for:

- "New size: X; reason: cpu resource utilization"
- Scale up/down decisions with timestamps

### Step 6: Configure better stabilization

```bash
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cpu-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpu-app
  minReplicas: 2
  maxReplicas: 10
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 600  # 10 minutes
      policies:
      - type: Percent
        value: 10      # Scale down max 10% at a time
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Percent
        value: 100     # Can double capacity
        periodSeconds: 15
      - type: Pods
        value: 4       # Or add 4 pods
        periodSeconds: 15
      selectPolicy: Max  # Use whichever adds more
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50  # More reasonable
EOF

```

### Step 7: Test multiple metrics (conflict scenario)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: multi-metric-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpu-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
EOF

```

With multiple metrics, HPA uses the one recommending the MOST replicas.
This can cause unexpected scaling if one metric spikes.

**Check which metric is driving decisions:**

```bash
kubectl describe hpa multi-metric-hpa | grep -A 10 "Metrics:"

```

### Step 8: Custom metrics example (conceptual)

```yaml
# Requires prometheus-adapter or custom metrics API
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: custom-metric-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpu-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: 100  # Scale to keep ~100 RPS per pod
  - type: External
    external:
      metric:
        name: sqs_queue_length
        selector:
          matchLabels:
            queue: orders
      target:
        type: Value
        value: 1000  # Scale if queue exceeds 1000 messages

```

### Step 9: Simulate real-world traffic pattern

```bash
# Create a load generator deployment
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: traffic-sim
spec:
  replicas: 0
  selector:
    matchLabels:
      app: traffic-sim
  template:
    metadata:
      labels:
        app: traffic-sim
    spec:
      containers:
      - name: load
        image: busybox
        command: ['sh', '-c', 'while true; do wget -q -O- <http://cpu-app>; sleep 0.1; done']
EOF

```

**Simulate traffic spikes:**

```bash
# Spike traffic
kubectl scale deployment traffic-sim --replicas=10
sleep 30
kubectl get hpa cpu-app-hpa

# Reduce traffic
kubectl scale deployment traffic-sim --replicas=2
sleep 60
kubectl get hpa cpu-app-hpa

# Another spike
kubectl scale deployment traffic-sim --replicas=10

```

Watch for thrashing patterns!

Key Observations

✅ **Asymmetric scaling** - scale-up is fast, scale-down is slow (by design)
✅ **Stabilization windows** - prevent oscillation between states
✅ **Multiple metrics** - highest replica count wins
✅ **Resource requests** - directly affect utilization percentages

Production Patterns

**Conservative web app HPA:**

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300
    policies:
    - type: Percent
      value: 10
      periodSeconds: 60
  scaleUp:
    stabilizationWindowSeconds: 0
    policies:
    - type: Percent
      value: 50
      periodSeconds: 30

```

**Batch processing HPA (scale to zero with KEDA):**

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: batch-processor
spec:
  scaleTargetRef:
    name: batch-processor
  minReplicaCount: 0
  maxReplicaCount: 100
  triggers:
  - type: rabbitmq
    metadata:
      queueName: jobs
      queueLength: "5"

```

Cleanup

```bash
kubectl delete hpa cpu-app-hpa multi-metric-hpa 2>/dev/null
kubectl delete deployment cpu-app traffic-sim 2>/dev/null
kubectl delete service cpu-app 2>/dev/null

```

---

Next: Day 13 - PodDisruptionBudget Blocking Node Drains