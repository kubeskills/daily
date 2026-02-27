## Day 70: Autoscaling Failures - HPA & VPA Gone Wrong

### Email Copy

**Subject:** Day 70: Autoscaling Failures - When Pods Won't Scale

THE IDEA:
Configure HPA and VPA and watch autoscaling fail. Pods don't scale despite high CPU,
metrics unavailable, and conflicting VPA/HPA configurations fight each other.

THE SETUP:
Deploy HPA without metrics-server, set impossible target utilizations, run VPA and
HPA together, and test what breaks when autoscaling fails.

WHAT I LEARNED:
- HPA requires metrics-server for CPU/memory metrics
- Target utilization is percentage of requests (not limits)
- HPA and VPA conflict on CPU/memory (don't use together)
- Scale-up delay prevents flapping
- Custom metrics require adapters (Prometheus, etc.)
- Min/max replicas prevent runaway scaling

WHY IT MATTERS:
Autoscaling failures cause:
- Load spikes crash pods (didn't scale in time)
- Wasted resources (scaled too high)
- Flapping replicas (constant scale up/down)
- Outages when pods can't handle traffic

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-070

Tomorrow: Operator failures and custom controller issues.

---
Chad


---

## Killercoda Lab Instructions



## Step 1: Check if metrics-server is installed

```bash
# Check for metrics-server
kubectl get deployment -n kube-system metrics-server 2>/dev/null && echo "Metrics-server installed" || echo "Metrics-server NOT installed"

# Try to get pod metrics
kubectl top pods 2>&1 || echo "Metrics not available"

# Try to get node metrics
kubectl top nodes 2>&1 || echo "Node metrics not available"
```

## Step 2: Deploy application for testing

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "200m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
spec:
  selector:
    app: php-apache
  ports:
  - port: 80
EOF

kubectl wait --for=condition=Ready pod -l app=php-apache --timeout=60s
```

## Step 3: Create HPA without metrics-server

```bash
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
EOF

# Check HPA status
sleep 10
kubectl get hpa php-apache-hpa
kubectl describe hpa php-apache-hpa
```

Shows "unable to get metrics" if metrics-server missing!

## Step 4: Test HPA without resource requests

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: no-requests
spec:
  replicas: 1
  selector:
    matchLabels:
      app: no-requests
  template:
    metadata:
      labels:
        app: no-requests
    spec:
      containers:
      - name: app
        image: nginx
        # No resource requests!
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: no-requests-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: no-requests
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
EOF

sleep 10
kubectl describe hpa no-requests-hpa | grep -A 5 "Conditions:"
```

HPA fails - needs resource requests to calculate utilization!

## Step 5: Test impossible target utilization

```bash
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: impossible-target
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 10  # Very low target
EOF

# Even idle pods exceed 10% utilization
sleep 10
kubectl get hpa impossible-target
```

Constantly tries to scale up!

## Step 6: Generate load to trigger scaling

```bash
# Generate load
kubectl run -it load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done" &
LOAD_PID=$!

# Watch HPA scale
kubectl get hpa php-apache-hpa -w &
WATCH_PID=$!

sleep 60

# Stop load and watch
kill $LOAD_PID 2>/dev/null
sleep 30
kill $WATCH_PID 2>/dev/null
```

## Step 7: Test min/max replica limits

```bash
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: limited-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 5  # Always at least 5
  maxReplicas: 5  # Never more than 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
EOF

# Can't scale below or above limits
kubectl get deployment php-apache
sleep 30
kubectl get deployment php-apache
```

## Step 8: Test scale-down behavior

```bash
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: slow-scaledown
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scale down
      policies:
      - type: Percent
        value: 50  # Scale down by max 50% at once
        periodSeconds: 60
EOF

echo "Scale-down will be slow (5 minute stabilization window)"
```

## Step 9: Test VPA configuration

```bash
# VPA requires VPA controller (may not be installed)
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: php-apache-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  updatePolicy:
    updateMode: "Auto"  # Automatically restart pods with new resources
EOF

sleep 10
kubectl get vpa php-apache-vpa 2>&1 || echo "VPA not installed"
```

## Step 10: Test conflicting HPA and VPA

```bash
# Running both HPA and VPA on same deployment
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: conflict-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
---
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: conflict-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      controlledResources: ["cpu", "memory"]
EOF

echo "HPA and VPA both controlling CPU = conflict!"
echo "Recommendation: Use VPA for memory only, HPA for replicas"
```

## Step 11: Test custom metrics (without adapter)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: custom-metric-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
EOF

sleep 10
kubectl describe hpa custom-metric-hpa
```

Shows "unable to get metric" - needs custom metrics adapter!

## Step 12: Test scaling delay and cooldown

```bash
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: delayed-scaling
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60  # Wait 1 min before scale up
      policies:
      - type: Percent
        value: 100  # Double at once
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scale down
EOF

echo "Scaling delays prevent flapping but can be slow to react"
```

## Step 13: Test HPA with memory metrics

```bash
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: memory-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
EOF

kubectl describe hpa memory-hpa
```

## Step 14: Test multiple metrics

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
    name: php-apache
  minReplicas: 1
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
        averageUtilization: 80
EOF

# Scales based on whichever metric is higher
kubectl describe hpa multi-metric-hpa
```

## Step 15: Diagnose autoscaling issues

```bash
cat > /tmp/autoscaling-diagnosis.sh << 'EOF'
#!/bin/bash
echo "=== Autoscaling Diagnosis ==="

echo -e "\n1. HPA Status:"
kubectl get hpa -A -o custom-columns=\
NAMESPACE:.metadata.namespace,\
NAME:.metadata.name,\
TARGETS:.status.currentMetrics[0].resource.current.averageUtilization,\
MINPODS:.spec.minReplicas,\
MAXPODS:.spec.maxReplicas,\
REPLICAS:.status.currentReplicas

echo -e "\n2. HPAs Unable to Get Metrics:"
kubectl get hpa -A -o json | jq -r '
  .items[] |
  select(.status.conditions[]? | .type == "ScalingActive" and .status == "False") |
  "\(.metadata.namespace)/\(.metadata.name): \(.status.conditions[] | select(.type == "ScalingActive").message)"
'

echo -e "\n3. HPAs Without Resource Requests:"
kubectl get hpa -A -o json | jq -r '
  .items[] |
  .metadata.namespace as $ns |
  .spec.scaleTargetRef.name as $target |
  select(.spec.metrics[]?.resource.name == "cpu" or .spec.metrics[]?.resource.name == "memory") |
  "\($ns)/\(.metadata.name) targets \($target)"
' | while read line; do
  ns=$(echo $line | cut -d/ -f1)
  target=$(echo $line | awk '{print $NF}')
  requests=$(kubectl get deployment -n $ns $target -o jsonpath='{.spec.template.spec.containers[0].resources.requests}' 2>/dev/null)
  if [ -z "$requests" ] || [ "$requests" == "null" ]; then
    echo "$line: No resource requests!"
  fi
done

echo -e "\n4. VPAs (if installed):"
kubectl get vpa -A 2>/dev/null || echo "VPA not installed"

echo -e "\n5. Metrics Server Status:"
kubectl get deployment -n kube-system metrics-server 2>/dev/null || echo "Metrics-server not found"

echo -e "\n6. Recent HPA Events:"
kubectl get events -A --sort-by='.lastTimestamp' | grep -i "horizontalpodautoscaler\|hpa" | tail -10

echo -e "\n7. Common Autoscaling Issues:"
echo "   - No metrics-server: HPA can't get CPU/memory"
echo "   - No resource requests: Can't calculate utilization"
echo "   - Target too low: Constantly scaling up"
echo "   - HPA + VPA conflict: Both try to control same resource"
echo "   - Slow metrics: Delayed scaling decisions"
echo "   - Min = Max: Can't autoscale at all"
EOF

chmod +x /tmp/autoscaling-diagnosis.sh
/tmp/autoscaling-diagnosis.sh
```

## Key Observations

✅ **Metrics-server required** - for CPU/memory metrics
✅ **Resource requests needed** - to calculate utilization percentage
✅ **Target = % of requests** - not limits
✅ **HPA vs VPA** - don't use both for CPU/memory
✅ **Stabilization window** - prevents flapping
✅ **Min/max replicas** - prevent runaway scaling

## Production Patterns

**Production-ready HPA:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: production-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: production-app
  minReplicas: 3  # Always have redundancy
  maxReplicas: 50  # Prevent cost explosion
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Conservative target
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 50  # Grow by 50% at once
        periodSeconds: 60
      - type: Pods
        value: 10  # Or add 10 pods
        periodSeconds: 60
      selectPolicy: Max  # Use whichever scales faster
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min
      policies:
      - type: Percent
        value: 10  # Shrink by 10% at once
        periodSeconds: 60
      selectPolicy: Min  # Use conservative policy
```

**VPA for memory (HPA for replicas):**
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      controlledResources: ["memory"]  # Only memory!
      minAllowed:
        memory: "128Mi"
      maxAllowed:
        memory: "4Gi"
```

**Custom metrics with Prometheus:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: custom-metrics-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  - type: Object
    object:
      metric:
        name: requests_per_second
      describedObject:
        apiVersion: v1
        kind: Service
        name: api-server
      target:
        type: Value
        value: "10000"
```

**Monitoring autoscaling:**
```yaml
# Prometheus alerts
- alert: HPAMaxedOut
  expr: |
    kube_horizontalpodautoscaler_status_current_replicas ==
    kube_horizontalpodautoscaler_spec_max_replicas
  for: 15m
  annotations:
    summary: "HPA {{ $labels.namespace }}/{{ $labels.horizontalpodautoscaler }} at max replicas"

- alert: HPAScalingDisabled
  expr: |
    kube_horizontalpodautoscaler_status_condition{condition="ScalingActive",status="false"} == 1
  for: 10m
  annotations:
    summary: "HPA {{ $labels.namespace }}/{{ $labels.horizontalpodautoscaler }} unable to scale"

- alert: FrequentScaling
  expr: |
    rate(kube_horizontalpodautoscaler_status_desired_replicas[5m]) > 0.1
  annotations:
    summary: "HPA {{ $labels.namespace }}/{{ $labels.horizontalpodautoscaler }} scaling too frequently"
```

## Cleanup

```bash
kubectl delete hpa --all
kubectl delete vpa --all 2>/dev/null
kubectl delete deployment php-apache no-requests
kubectl delete service php-apache
rm -f /tmp/autoscaling-diagnosis.sh
```

---
Next: Day 71 - Operator Failures and Custom Controller Issues
