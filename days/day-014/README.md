# Day 14: ResourceQuota - Deployment Denied

## THE IDEA:
Set a namespace ResourceQuota and watch deployments fail silently. The
ReplicaSet creates zero pods with no obvious error - until you know where
to look.

## THE SETUP:
Create a tight ResourceQuota, deploy an app without resource requests, and
discover why your pods never appear. Then explore LimitRanges for automatic
defaults.

## WHAT I LEARNED:
- ResourceQuota blocks pod creation, but Deployment shows no error
- Must check ReplicaSet events to see quota violations
- LimitRange can inject default requests/limits automatically
- Quota scopes let you set different limits for different pod priorities

## WHY IT MATTERS:
Quota issues cause:
- "My deployment succeeded but no pods appeared"
- Silent failures in CI/CD pipelines
- Namespace resource starvation from one team

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-014

Week 2 complete! Next week: Ingress misconfigurations, RBAC lockouts, and
admission webhook chaos.

---

# Killercoda Lab Instructions


## Step 1: Create a namespace with ResourceQuota

```
kubectl create namespace quota-test

cat <<EOF | kubectl apply -n quota-test -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: strict-quota
spec:
  hard:
    requests.cpu: "500m"
    requests.memory: "512Mi"
    limits.cpu: "1"
    limits.memory: "1Gi"
    pods: "5"
    persistentvolumeclaims: "2"
    services.loadbalancers: "0"
EOF

```

**Check quota status:**

```bash
kubectl get resourcequota -n quota-test
kubectl describe resourcequota strict-quota -n quota-test

```

Step 2: Deploy without resource requests (fails silently)

```bash
cat <<EOF | kubectl apply -n quota-test -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: no-resources
spec:
  replicas: 3
  selector:
    matchLabels:
      app: no-resources
  template:
    metadata:
      labels:
        app: no-resources
    spec:
      containers:
      - name: app
        image: nginx
        # No resources defined!
EOF

```

**Check deployment:**

```bash
kubectl get deployment -n quota-test no-resources

```

Shows 0/3 ready. No obvious error!

**Check pods:**

```bash
kubectl get pods -n quota-test

```

No pods! Where did they go?

Step 3: Find the real error

```bash
kubectl get replicaset -n quota-test

```

ReplicaSet exists but DESIRED != READY.

**Check ReplicaSet events:**

```bash
kubectl describe replicaset -n quota-test | grep -A 10 Events

```

ERROR: "failed quota: strict-quota: must specify requests.cpu, requests.memory"

When quota exists, ALL pods must specify requests for quotaed resources.

Step 4: Fix by adding resource requests

```bash
cat <<EOF | kubectl apply -n quota-test -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: with-resources
spec:
  replicas: 3
  selector:
    matchLabels:
      app: with-resources
  template:
    metadata:
      labels:
        app: with-resources
    spec:
      containers:
      - name: app
        image: nginx
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
EOF

```

**Check pods:**

```bash
kubectl get pods -n quota-test -l app=with-resources

```

Pods are running!

**Check quota usage:**

```bash
kubectl describe resourcequota strict-quota -n quota-test

```

Shows used vs hard limits.

Step 5: Exceed the quota

```bash
kubectl scale deployment with-resources -n quota-test --replicas=10

```

**Check results:**

```bash
kubectl get deployment with-resources -n quota-test
kubectl get replicaset -n quota-test -l app=with-resources

```

Only 5 pods created (hit pod limit). Check events:

```bash
kubectl describe replicaset -n quota-test -l app=with-resources | tail -15

```

Error: "exceeded quota: strict-quota, requested: pods=1, used: pods=5, limited: pods=5"

Step 6: Add LimitRange for automatic defaults

```bash
cat <<EOF | kubectl apply -n quota-test -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
  - default:
      cpu: 200m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: 500m
      memory: 512Mi
    min:
      cpu: 50m
      memory: 64Mi
    type: Container
EOF

```

**Delete the broken deployment and recreate without resources:**

```bash
kubectl delete deployment no-resources -n quota-test

cat <<EOF | kubectl apply -n quota-test -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auto-limits
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auto-limits
  template:
    metadata:
      labels:
        app: auto-limits
    spec:
      containers:
      - name: app
        image: nginx
        # No resources defined - LimitRange will inject them!
EOF

```

**Check pod resources:**

```bash
kubectl get pod -n quota-test -l app=auto-limits -o jsonpath='{.items[0].spec.containers[0].resources}'
echo ""

```

LimitRange injected default requests and limits!

Step 7: LimitRange violations

```bash
cat <<EOF | kubectl apply -n quota-test -f -
apiVersion: v1
kind: Pod
metadata:
  name: too-big
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: 1000m  # Exceeds LimitRange max!
        memory: 1Gi
EOF

```

ERROR: "cpu max limit to be 500m" - LimitRange blocks oversized pods.

Step 8: Quota scopes (priority-based)

```bash
cat <<EOF | kubectl apply -n quota-test -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: high-priority-quota
spec:
  hard:
    cpu: "2"
    memory: "2Gi"
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["high"]
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: low-priority-quota
spec:
  hard:
    cpu: "500m"
    memory: "512Mi"
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["low"]
EOF

```

Different quotas for different priority classes!

Step 9: Count-based quotas

```bash
cat <<EOF | kubectl apply -n quota-test -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: count-quota
spec:
  hard:
    count/deployments.apps: "3"
    count/services: "5"
    count/secrets: "10"
    count/configmaps: "10"
EOF

```

**Try to exceed:**

```bash
for i in {1..5}; do kubectl create deployment test-$i --image=nginx -n quota-test; done

```

Fourth deployment fails!

Step 10: Cross-namespace resource tracking

```bash
# View all quota usage across namespaces
kubectl get resourcequota --all-namespaces

```

Key Observations

✅ **Silent failures** - Deployment succeeds, ReplicaSet fails to create pods
✅ **Must check ReplicaSet events** - quota errors don't bubble up to Deployment
✅ **LimitRange defaults** - auto-inject resources when quota requires them
✅ **Scoped quotas** - different limits for different pod classes
✅ **Count quotas** - limit number of objects, not just resources

Production Patterns

**Team namespace quota:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "50"
    services: "20"
    persistentvolumeclaims: "10"
    secrets: "50"
    configmaps: "50"

```

**Prevent LoadBalancer sprawl:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: no-loadbalancers
spec:
  hard:
    services.loadbalancers: "0"
    services.nodeports: "0"

```

**LimitRange for safety:**

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
spec:
  limits:
  - type: Container
    max:
      cpu: "2"
      memory: "4Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
  - type: PersistentVolumeClaim
    max:
      storage: "50Gi"
    min:
      storage: "1Gi"

```

Cleanup

```bash
kubectl delete namespace quota-test

```

---

🎉 Week 2 Complete! 14 days of Kubernetes failure modes mastered.