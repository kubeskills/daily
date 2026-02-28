## Day 65: Image Pull Failures - Registry Nightmares

### Email Copy

**Subject:** Day 65: Image Pull Failures - When Images Won't Download

THE IDEA:
Deploy pods with broken image references and watch them fail. ImagePullBackOff errors,
wrong registry credentials, rate limiting from Docker Hub, and missing image tags.

THE SETUP:
Use nonexistent images, wrong registry URLs, invalid credentials, hit rate limits,
and test what breaks when container images can't be pulled.

WHAT I LEARNED:
- ImagePullBackOff = exponential backoff after pull failures
- Docker Hub rate limits: 100 pulls/6h (anonymous), 200/6h (authenticated)
- imagePullSecrets required for private registries
- Image pull happens on the node where pod schedules
- Latest tag is mutable (can change without notice)

WHY IT MATTERS:
Image pull failures cause:
- Pods stuck in ImagePullBackOff indefinitely
- Deployments rollout stuck (new version won't start)
- Rate limit exhaustion blocks entire cluster
- Nodes download same image repeatedly (no caching)

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-065

Tomorrow: Resource quota violations and limit ranges.

---
Chad


---

## Killercoda Lab Instructions



## Step 1: Test nonexistent image

```bash
# Try to pull image that doesn't exist
kubectl run nonexistent --image=nginx:nonexistent-tag-12345

# Check status
sleep 10
kubectl get pod nonexistent
kubectl describe pod nonexistent | grep -A 10 "Events:"
```

Shows "ErrImagePull" then "ImagePullBackOff"!

## Step 2: Check ImagePullBackOff backoff timing

```bash
# Watch backoff progression
kubectl get pod nonexistent -w &
WATCH_PID=$!

sleep 30
kill $WATCH_PID 2>/dev/null

# Check events to see backoff
kubectl get events --field-selector involvedObject.name=nonexistent --sort-by='.lastTimestamp' | tail -10
```

Backoff: 10s, 20s, 40s, 80s, 160s, max 5 minutes

## Step 3: Test wrong registry

```bash
kubectl run wrong-registry --image=wrong-registry.example.com/nginx:latest

sleep 10
kubectl describe pod wrong-registry | grep -A 5 "Failed"
```

Shows DNS or connection error!

## Step 4: Test private registry without credentials

```bash
# Try to pull from private registry (will fail)
kubectl run private-image --image=private-registry.example.com/myapp:v1.0

sleep 10
kubectl describe pod private-image | grep -A 5 "Events:"
```

## Step 5: Create imagePullSecret

```bash
# Create docker registry secret
kubectl create secret docker-registry my-registry-secret \
  --docker-server=private-registry.example.com \
  --docker-username=testuser \
  --docker-password=testpass \
  --docker-email=test@example.com

# Check secret
kubectl get secret my-registry-secret -o yaml
```

## Step 6: Use imagePullSecret in pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: with-pull-secret
spec:
  imagePullSecrets:
  - name: my-registry-secret
  containers:
  - name: app
    image: private-registry.example.com/myapp:v1.0
EOF

# Still fails (registry doesn't exist), but shows proper auth attempt
sleep 10
kubectl describe pod with-pull-secret | grep -A 5 "Events:"
```

## Step 7: Test Docker Hub rate limiting

```bash
# Check current pull count (if accessible)
echo "Docker Hub rate limits:"
echo "- Anonymous: 100 pulls per 6 hours"
echo "- Authenticated: 200 pulls per 6 hours"
echo "- Rate limit is per IP address"

# Simulate hitting rate limit
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rate-limit-test
spec:
  replicas: 10
  selector:
    matchLabels:
      app: rate-test
  template:
    metadata:
      labels:
        app: rate-test
    spec:
      containers:
      - name: app
        image: nginx:latest  # Public image
        command: ['sleep', '3600']
EOF

kubectl wait --for=condition=Ready pod -l app=rate-test --timeout=120s

# Delete and recreate to trigger new pulls
kubectl delete deployment rate-limit-test
sleep 5
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rate-limit-test
spec:
  replicas: 10
  selector:
    matchLabels:
      app: rate-test
  template:
    metadata:
      labels:
        app: rate-test
    spec:
      containers:
      - name: app
        image: nginx:alpine  # Different image
EOF

# Check for rate limit errors
sleep 30
kubectl get events --sort-by='.lastTimestamp' | grep -i "rate\|limit\|throttle" | tail -5
```

## Step 8: Test image pull policy

```bash
# Always pull (ignores local cache)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: always-pull
spec:
  containers:
  - name: app
    image: nginx:latest
    imagePullPolicy: Always  # Always pulls, even if cached
EOF

# Never pull (fails if not cached)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: never-pull
spec:
  containers:
  - name: app
    image: nginx:nonexistent-987654
    imagePullPolicy: Never  # Never pulls
EOF

sleep 10
kubectl describe pod never-pull | grep -A 5 "Events:"
```

## Step 9: Test missing image tag (defaults to :latest)

```bash
kubectl run no-tag --image=nginx  # Implicitly :latest

kubectl get pod no-tag -o jsonpath='{.spec.containers[0].image}'
echo ""
```

Shows "nginx" becomes "nginx:latest"

## Step 10: Test image digest pinning

```bash
# Use image digest for immutability
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: digest-pin
spec:
  containers:
  - name: app
    image: nginx@sha256:2d4c0b9f9d8f8b7a8c8e8f8a8b8c8d8e8f8a8b8c8d8e8f8a8b8c8d8e8f8a8b8c
    # Digest ensures exact image version
EOF

# Check if image exists with that digest
sleep 10
kubectl describe pod digest-pin | grep -A 5 "Events:"
```

## Step 11: Test concurrent image pulls

```bash
# Deploy many pods pulling same image simultaneously
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: concurrent-pulls
spec:
  replicas: 20
  selector:
    matchLabels:
      app: concurrent
  template:
    metadata:
      labels:
        app: concurrent
    spec:
      containers:
      - name: app
        image: redis:7-alpine
        imagePullPolicy: Always
EOF

# Watch pull progress
kubectl get pods -l app=concurrent -w
```

## Step 12: Test image pull timeout

```bash
# Simulate slow registry (concept)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: slow-pull
spec:
  containers:
  - name: app
    image: slow-registry.example.com/large-image:latest
    # Large images can timeout
EOF

sleep 15
kubectl describe pod slow-pull | grep -A 10 "Events:"
```

## Step 13: Test ServiceAccount with imagePullSecrets

```bash
# Create ServiceAccount with pull secret
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: registry-sa
imagePullSecrets:
- name: my-registry-secret
EOF

# Use ServiceAccount in pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: sa-pull-secret
spec:
  serviceAccountName: registry-sa
  containers:
  - name: app
    image: private-registry.example.com/app:v1
EOF

# ServiceAccount automatically provides pull secret
kubectl describe pod sa-pull-secret | grep -A 2 "Image Pull Secrets"
```

## Step 14: Check node image cache

```bash
# List images on node (if accessible)
echo "Images cached on nodes:"
kubectl get nodes -o wide

# In production, you would check:
# crictl images (for containerd)
# docker images (for docker)

echo "Image caching:"
echo "- First pull: downloads from registry"
echo "- Subsequent pulls: uses local cache (unless imagePullPolicy: Always)"
echo "- Cache saved in /var/lib/containerd or /var/lib/docker"
```

## Step 15: Diagnose image pull issues

```bash
cat > /tmp/image-diagnosis.sh << 'EOF'
#!/bin/bash
echo "=== Image Pull Diagnosis ==="

echo -e "\n1. Pods in ImagePullBackOff:"
kubectl get pods -A --field-selector status.phase=Pending -o json | \
  jq -r '.items[] | select(.status.containerStatuses[]?.state.waiting?.reason == "ImagePullBackOff") | "\(.metadata.namespace)/\(.metadata.name)"'

echo -e "\n2. Recent Image Pull Errors:"
kubectl get events -A --sort-by='.lastTimestamp' | grep -i "pull\|image" | grep -i "error\|fail" | tail -10

echo -e "\n3. Registry Credentials:"
kubectl get secrets -A --field-selector type=kubernetes.io/dockerconfigjson -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name

echo -e "\n4. Pods with ImagePullSecrets:"
kubectl get pods -A -o json | \
  jq -r '.items[] | select(.spec.imagePullSecrets != null) | "\(.metadata.namespace)/\(.metadata.name): \(.spec.imagePullSecrets[].name)"' | head -10

echo -e "\n5. Image Pull Policy Distribution:"
kubectl get pods -A -o json | \
  jq -r '.items[].spec.containers[].imagePullPolicy' | sort | uniq -c

echo -e "\n6. Images Using :latest Tag:"
kubectl get pods -A -o json | \
  jq -r '.items[] | select(.spec.containers[].image | endswith(":latest")) | "\(.metadata.namespace)/\(.metadata.name)"' | head -10

echo -e "\n7. Common Image Pull Issues:"
echo "   - Nonexistent tag: Check image exists in registry"
echo "   - Wrong registry URL: Verify registry domain"
echo "   - Missing credentials: Add imagePullSecret"
echo "   - Rate limited: Use authenticated pulls or mirror"
echo "   - Network issues: Check node connectivity to registry"
echo "   - Large image timeout: Increase timeout or use smaller images"
EOF

chmod +x /tmp/image-diagnosis.sh
/tmp/image-diagnosis.sh
```

## Key Observations

✅ **ImagePullBackOff** - exponential backoff after failures
✅ **imagePullSecrets** - required for private registries
✅ **Rate limits** - Docker Hub limits unauthenticated pulls
✅ **imagePullPolicy** - controls when to pull from registry
✅ **:latest tag** - mutable, can change without notice
✅ **Image caching** - nodes cache pulled images

## Production Patterns

**Secure image pull configuration:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: production
imagePullSecrets:
- name: production-registry
---
apiVersion: v1
kind: Secret
metadata:
  name: production-registry
  namespace: production
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3
  template:
    spec:
      serviceAccountName: app-sa
      containers:
      - name: app
        image: myregistry.example.com/myapp:v1.2.3
        imagePullPolicy: IfNotPresent  # Use cache when available
```

**Image pinning with digest:**
```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        # Pin to exact digest for immutability
        image: nginx@sha256:10d1f5b58f74683ad34eb29287e07dab1e90f10af243f151bb50aa5dbb4d62ee
```

**Registry mirror configuration:**
```yaml
# For nodes to use registry mirror (reduce rate limit hits)
# /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".registry.mirrors]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
    endpoint = ["https://registry-mirror.example.com"]
```

**Image pull monitoring:**
```yaml
# Prometheus alerts
- alert: ImagePullBackOff
  expr: |
    kube_pod_container_status_waiting_reason{reason="ImagePullBackOff"} > 0
  for: 10m
  annotations:
    summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} stuck pulling image"

- alert: HighImagePullFailureRate
  expr: |
    rate(kubelet_runtime_operations_errors_total{operation_type="pull_image"}[5m]) > 0.1
  annotations:
    summary: "High image pull failure rate on {{ $labels.node }}"

- alert: RegistryUnreachable
  expr: |
    probe_success{job="registry-monitor"} == 0
  for: 5m
  annotations:
    summary: "Container registry unreachable"
```

**Pre-pull critical images:**
```yaml
# DaemonSet to pre-pull images on all nodes
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: image-prepull
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: image-prepull
  template:
    metadata:
      labels:
        app: image-prepull
    spec:
      initContainers:
      - name: prepull-app
        image: myregistry.example.com/myapp:v1.2.3
        command: ['sh', '-c', 'echo Image pulled']
      - name: prepull-db
        image: postgres:15-alpine
        command: ['sh', '-c', 'echo Image pulled']
      containers:
      - name: pause
        image: registry.k8s.io/pause:3.9
```

## Cleanup

```bash
kubectl delete pod nonexistent wrong-registry private-image with-pull-secret always-pull never-pull no-tag digest-pin slow-pull sa-pull-secret
kubectl delete deployment rate-limit-test concurrent-pulls
kubectl delete secret my-registry-secret
kubectl delete serviceaccount registry-sa
rm -f /tmp/image-diagnosis.sh
```

---
Next: Day 66 - Resource Quota Violations and Limit Ranges
