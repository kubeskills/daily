## Day 66: Resource Quotas - Hitting the Limits

### Email Copy

**Subject:** Day 66: Resource Quotas - When You Can't Create Anything

THE IDEA:
Exhaust namespace resource quotas and watch all new resource creation fail. Pods
rejected, services can't be created, and deployments stuck at 0/3 ready.

THE SETUP:
Create restrictive ResourceQuotas, hit pod limits, exhaust CPU/memory budgets,
and test what breaks when quotas are exceeded.

WHAT I LEARNED:
- ResourceQuota limits resources per namespace
- Pods rejected if quota exceeded
- LimitRange provides defaults when not specified
- CPU/memory requests count toward quota (not limits)
- Some resources counted: pods, services, PVCs, ConfigMaps

WHY IT MATTERS:
Quota violations cause:
- Deployments can't scale (quota exhausted)
- New services rejected (too many services)
- CI/CD pipelines fail (can't create pods)
- Team blocked until quota increased

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-066

Tomorrow: Init container failures and sidecar issues.

---
Chad


---

## Killercoda Lab Instructions



## Step 1: Create namespace with ResourceQuota

```bash
kubectl create namespace quota-test

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: basic-quota
  namespace: quota-test
spec:
  hard:
    requests.cpu: "2"
    requests.memory: "2Gi"
    limits.cpu: "4"
    limits.memory: "4Gi"
    pods: "5"
    services: "3"
    persistentvolumeclaims: "2"
    configmaps: "5"
    secrets: "5"
EOF

# Check quota
kubectl describe resourcequota basic-quota -n quota-test
```

## Step 2: Deploy pods within quota

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: within-quota
  namespace: quota-test
spec:
  replicas: 3
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - name: app
        image: nginx
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
EOF

kubectl wait --for=condition=Ready pod -n quota-test -l app=test --timeout=60s

# Check quota usage
kubectl describe resourcequota basic-quota -n quota-test
```

Shows: Used: requests.cpu: 1500m/2, requests.memory: 1536Mi/2Gi, pods: 3/5

## Step 3: Try to exceed pod quota

```bash
# Try to scale beyond pod limit
kubectl scale deployment within-quota -n quota-test --replicas=6

sleep 10

# Check actual replicas
kubectl get deployment within-quota -n quota-test

# Check events
kubectl get events -n quota-test --sort-by='.lastTimestamp' | grep -i quota | tail -5
```

Shows "exceeded quota" error!

## Step 4: Try to exceed CPU quota

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cpu-hog
  namespace: quota-test
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "1000m"  # Would exceed remaining 500m
        memory: "256Mi"
EOF

# Pod rejected
sleep 5
kubectl get pod cpu-hog -n quota-test 2>&1 || echo "Pod creation failed"
kubectl get events -n quota-test | grep cpu-hog
```

## Step 5: Test quota without resource requests

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-requests
  namespace: quota-test
spec:
  containers:
  - name: app
    image: nginx
    # No resource requests specified!
EOF

# Rejected because quota requires requests
sleep 5
kubectl describe pod no-requests -n quota-test 2>&1 | grep -A 5 "Events:"
```

Shows "must specify requests" error!

## Step 6: Add LimitRange to provide defaults

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: quota-test
spec:
  limits:
  - default:  # Default limits
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:  # Default requests
      cpu: "100m"
      memory: "128Mi"
    type: Container
EOF

# Now pod without requests can be created
kubectl delete pod no-requests -n quota-test 2>/dev/null
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-requests
  namespace: quota-test
spec:
  containers:
  - name: app
    image: nginx
    # LimitRange provides defaults
EOF

sleep 5
kubectl describe pod no-requests -n quota-test | grep -A 5 "Requests:"
```

## Step 7: Test service quota

```bash
# Create services up to limit
for i in 1 2 3; do
  kubectl create service clusterip svc-$i -n quota-test --tcp=80:80
done

# Try to create 4th service (exceeds quota of 3)
kubectl create service clusterip svc-4 -n quota-test --tcp=80:80 2>&1 || echo "Service creation failed - quota exceeded"

# Check quota usage
kubectl describe resourcequota basic-quota -n quota-test | grep services
```

<!-- Content was truncated. Please add the remaining steps. -->
