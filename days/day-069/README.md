## Day 69: Service Mesh Failures - Istio/Linkerd Gone Wrong

### Email Copy

**Subject:** Day 69: Service Mesh Failures - When mTLS Breaks Everything

THE IDEA:
Deploy a service mesh and watch traffic fail due to mTLS issues, sidecar injection
failures, and circuit breaker misconfigurations. Services can't communicate despite
being healthy.

THE SETUP:
Install Istio/Linkerd, break mutual TLS, disable sidecar injection, misconfigure
virtual services, and test what happens when the mesh fails.

WHAT I LEARNED:
- Sidecar proxy intercepts all pod traffic
- mTLS requires both client and server certificates
- Strict mode blocks non-mesh traffic completely
- Sidecar injection controlled by namespace labels
- Circuit breakers prevent cascading failures
- DestinationRule controls traffic policies

WHY IT MATTERS:
Service mesh failures cause:
- All service-to-service traffic blocked (mTLS issues)
- Pods missing sidecars can't communicate with mesh
- Circuit breakers trip unnecessarily (false positives)
- Complex debugging (traffic intercepted by proxy)

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-069

Tomorrow: Autoscaling failures with HPA and VPA.

---
Chad
Kube Daily - Week 11: Advanced Operations & Observability


---

## Killercoda Lab Instructions

## Step 1: Check for existing service mesh

```bash
# Check for Istio
kubectl get namespace istio-system 2>/dev/null && echo "Istio detected" || echo "Istio not installed"

# Check for Linkerd
kubectl get namespace linkerd 2>/dev/null && echo "Linkerd detected" || echo "Linkerd not installed"

# For this lab, we'll simulate mesh behavior
echo "Service mesh concepts demonstration"
```

## Step 2: Simulate sidecar injection

```bash
# Create namespace with sidecar injection enabled
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: mesh-demo
  labels:
    istio-injection: enabled
EOF

# Deploy service with automatic sidecar
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-a
  namespace: mesh-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: service-a
  template:
    metadata:
      labels:
        app: service-a
    spec:
      containers:
      - name: app
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: service-a
  namespace: mesh-demo
spec:
  selector:
    app: service-a
  ports:
  - port: 80
EOF

kubectl wait --for=condition=Ready pod -n mesh-demo -l app=service-a --timeout=60s

# Check container count (with sidecar would be 2+)
kubectl get pod -n mesh-demo -l app=service-a -o jsonpath='{.items[0].spec.containers[*].name}'
echo ""
```

## Step 3: Test pod without sidecar injection

```bash
# Create namespace WITHOUT injection
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: no-mesh
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-b
  namespace: no-mesh
spec:
  replicas: 1
  selector:
    matchLabels:
      app: service-b
  template:
    metadata:
      labels:
        app: service-b
    spec:
      containers:
      - name: app
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: service-b
  namespace: no-mesh
spec:
  selector:
    app: service-b
  ports:
  - port: 80
EOF

kubectl wait --for=condition=Ready pod -n no-mesh -l app=service-b --timeout=60s

# Only has main container (no sidecar)
kubectl get pod -n no-mesh -l app=service-b -o jsonpath='{.items[0].spec.containers[*].name}'
echo ""
```

## Step 4: Simulate mTLS strict mode

```bash
# In strict mTLS mode, only mesh traffic allowed
# Simulate by using NetworkPolicy

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: strict-mtls-simulation
  namespace: mesh-demo
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          istio-injection: enabled
    # Only allow traffic from other mesh-enabled namespaces
EOF

# Test connectivity from mesh namespace
kubectl run test-mesh -n mesh-demo --rm -it --image=busybox --restart=Never -- \
  wget -O- --timeout=2 http://service-a.mesh-demo 2>&1 | grep -i "connected\|200" || echo "Connection works (same mesh)"

# Test from non-mesh namespace (should fail with strict mode)
kubectl run test-no-mesh -n no-mesh --rm -it --image=busybox --restart=Never -- \
  wget -O- --timeout=2 http://service-a.mesh-demo 2>&1 || echo "Blocked by strict mTLS simulation"
```

## Step 5: Simulate VirtualService routing

```bash
# Create two versions of a service
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v1
  namespace: mesh-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-text=Version 1"]
        ports:
        - containerPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v2
  namespace: mesh-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
      version: v2
  template:
    metadata:
      labels:
        app: myapp
        version: v2
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-text=Version 2"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: mesh-demo
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 5678
EOF

kubectl wait --for=condition=Ready pod -n mesh-demo -l app=myapp --timeout=60s

# Without VirtualService, traffic load balances across all pods
for i in {1..10}; do
  kubectl run test-$i -n mesh-demo --rm -i --image=curlimages/curl --restart=Never -- \
    curl -s http://myapp.mesh-demo | grep Version
done
```

## Step 6: Simulate DestinationRule circuit breaker

```bash
# Simulate circuit breaker with deployment that fails
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flaky-service
  namespace: mesh-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: flaky
  template:
    metadata:
      labels:
        app: flaky
    spec:
      containers:
      - name: app
        image: busybox
        command:
        - sh
        - -c
        - |
          # 50% of requests fail
          while true; do
            nc -l -p 8080 -e sh -c 'if [ $((RANDOM % 2)) -eq 0 ]; then echo "HTTP/1.1 500 Error"; else echo "HTTP/1.1 200 OK"; fi; echo'
          done
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: flaky
  namespace: mesh-demo
spec:
  selector:
    app: flaky
  ports:
  - port: 80
    targetPort: 8080
EOF

kubectl wait --for=condition=Ready pod -n mesh-demo -l app=flaky --timeout=60s

# Test flaky service
echo "Testing flaky service (50% failure rate):"
for i in {1..10}; do
  kubectl run test-flaky-$i -n mesh-demo --rm -i --image=busybox --restart=Never -- \
    wget -O- --timeout=1 http://flaky.mesh-demo 2>&1 | grep -o "200\|500" || echo "Timeout"
done
```

## Step 7: Simulate sidecar resource issues

```bash
# Sidecar proxy consuming too many resources
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-starved
  namespace: mesh-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: starved
  template:
    metadata:
      labels:
        app: starved
    spec:
      containers:
      - name: app
        image: nginx
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
      # Simulate sidecar
      - name: sidecar-proxy
        image: envoyproxy/envoy:v1.28-latest
        command: ['sleep', '3600']
        resources:
          requests:
            cpu: "500m"  # Heavy sidecar!
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
EOF

# Check resource usage
sleep 20
kubectl top pod -n mesh-demo -l app=starved 2>/dev/null || echo "Metrics not available"
```

## Step 8: Test peer authentication issues

```bash
# Simulate certificate mismatch
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: mtls-config
  namespace: mesh-demo
data:
  config: |
    # Simulated mTLS configuration
    mode: STRICT
    # All traffic must use mTLS
    # Pods without valid certificates blocked
EOF

echo "In strict mTLS mode:"
echo "- Pods must have valid client certificates"
echo "- Server must verify client certificates"
echo "- Certificate expiry causes traffic failures"
echo "- Mismatched CA blocks communication"
```

## Step 9: Simulate retry and timeout issues

```bash
# Service with long response time
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: slow-service
  namespace: mesh-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: slow
  template:
    metadata:
      labels:
        app: slow
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-text=Slow response"]
        ports:
        - containerPort: 5678
        readinessProbe:
          httpGet:
            path: /
            port: 5678
          initialDelaySeconds: 30  # Slow to start
---
apiVersion: v1
kind: Service
metadata:
  name: slow
  namespace: mesh-demo
spec:
  selector:
    app: slow
  ports:
  - port: 80
    targetPort: 5678
EOF

# Without proper timeouts, requests hang
kubectl run test-timeout -n mesh-demo --rm -i --image=curlimages/curl --restart=Never -- \
  curl --max-time 5 http://slow.mesh-demo 2>&1 || echo "Request timed out"
```

## Step 10: Test traffic mirroring/shadowing

```bash
# Simulate traffic shadowing concept
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production
  namespace: mesh-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: prod
  template:
    metadata:
      labels:
        app: prod
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-text=Production"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: canary
  namespace: mesh-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: canary
  template:
    metadata:
      labels:
        app: canary
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-text=Canary (mirrored traffic)"]
EOF

echo "Traffic mirroring sends copy of live traffic to canary"
echo "Used for testing without impacting production"
```

## Step 11: Simulate egress gateway failure

```bash
# Egress traffic control
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-external
  namespace: mesh-demo
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}
    # Only allow internal traffic
EOF

# Test external access (blocked)
kubectl run test-egress -n mesh-demo --rm -it --image=busybox --restart=Never -- \
  wget -O- --timeout=2 https://google.com 2>&1 || echo "External traffic blocked"
```

## Step 12: Test observability without mesh

```bash
# Without service mesh, no automatic tracing
kubectl run trace-test -n mesh-demo --image=nginx

echo "Without service mesh:"
echo "- No automatic distributed tracing"
echo "- No service-to-service metrics"
echo "- No mutual TLS by default"
echo "- Manual observability instrumentation required"
```

## Step 13: Simulate sidecar crash

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sidecar-crash
  namespace: mesh-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: crash-test
  template:
    metadata:
      labels:
        app: crash-test
    spec:
      containers:
      - name: app
        image: nginx
        ports:
        - containerPort: 80
      # Simulate crashing sidecar
      - name: proxy
        image: busybox
        command: ['sh', '-c', 'sleep 5; exit 1']
EOF

sleep 15

# Main container running but sidecar crashed
kubectl get pod -n mesh-demo -l app=crash-test
kubectl describe pod -n mesh-demo -l app=crash-test | grep -A 5 "State:"
```

## Step 14: Test service mesh debugging

```bash
cat > /tmp/mesh-diagnosis.sh << 'EOF'
#!/bin/bash
echo "=== Service Mesh Diagnosis ==="

echo -e "\n1. Namespaces with Sidecar Injection:"
kubectl get namespaces -L istio-injection,linkerd.io/inject

echo -e "\n2. Pods with Multiple Containers (potential sidecars):"
kubectl get pods -A -o json | jq -r '
  .items[] |
  select((.spec.containers | length) > 1) |
  "\(.metadata.namespace)/\(.metadata.name): \(.spec.containers | length) containers"
' | head -10

echo -e "\n3. Pods Missing Expected Sidecar:"
kubectl get pods -n mesh-demo -o json | jq -r '
  .items[] |
  select((.spec.containers | length) == 1) |
  "\(.metadata.name): Only main container (no sidecar?)"
'

echo -e "\n4. NetworkPolicies (mTLS simulation):"
kubectl get networkpolicies -A

echo -e "\n5. Service Mesh Common Issues:"
echo "   - Missing sidecar: Namespace not labeled for injection"
echo "   - mTLS failure: Certificate issues or strict mode"
echo "   - Traffic blocked: NetworkPolicy or PeerAuthentication"
echo "   - High latency: Sidecar proxy overhead"
echo "   - Circuit breaker: Too many failures trigger break"
echo "   - Version conflict: Mismatched mesh version on nodes"
EOF

chmod +x /tmp/mesh-diagnosis.sh
/tmp/mesh-diagnosis.sh
```

## Step 15: Test mesh upgrade scenario

```bash
# Simulate mesh version mismatch
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: old-sidecar
  namespace: mesh-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: old-version
  template:
    metadata:
      labels:
        app: old-version
      annotations:
        sidecar.istio.io/proxyImage: "istio/proxyv2:1.16.0"
    spec:
      containers:
      - name: app
        image: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: new-sidecar
  namespace: mesh-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: new-version
  template:
    metadata:
      labels:
        app: new-version
      annotations:
        sidecar.istio.io/proxyImage: "istio/proxyv2:1.20.0"
    spec:
      containers:
      - name: app
        image: nginx
EOF

echo "Version mismatch can cause:"
echo "- Protocol incompatibilities"
echo "- Feature flag differences"
echo "- Certificate validation issues"
```

## Key Observations

✅ **Sidecar injection** - controlled by namespace labels
✅ **mTLS strict mode** - blocks non-mesh traffic
✅ **VirtualService** - controls routing between service versions
✅ **DestinationRule** - defines traffic policies (circuit breaker, TLS)
✅ **Sidecar overhead** - CPU/memory impact on every pod
✅ **Certificate lifecycle** - expiry breaks mTLS

## Production Patterns

**Istio installation:**
```bash
# Install Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# Install with demo profile
istioctl install --set profile=demo -y

# Enable sidecar injection
kubectl label namespace default istio-injection=enabled
```

**Secure namespace with mTLS:**
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # Require mTLS for all traffic
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  {}  # Deny all by default
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend
  namespace: production
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/frontend"]
    to:
    - operation:
        methods: ["GET", "POST"]
```

**Traffic management:**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v2
      weight: 10
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        maxRequestsPerConnection: 2
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

**Monitoring mesh health:**
```yaml
# Prometheus alerts
- alert: SidecarInjectionFailed
  expr: |
    sum(rate(sidecar_injection_failure_total[5m])) > 0
  annotations:
    summary: "Sidecar injection failures detected"

- alert: MutualTLSConfigurationError
  expr: |
    sum(rate(pilot_conflict_inbound_listener[5m])) > 0
  annotations:
    summary: "mTLS configuration conflicts"

- alert: HighProxyResourceUsage
  expr: |
    container_memory_working_set_bytes{container="istio-proxy"} /
    container_spec_memory_limit_bytes{container="istio-proxy"} > 0.9
  annotations:
    summary: "Proxy sidecar using > 90% memory"
```

## Cleanup

```bash
kubectl delete namespace mesh-demo no-mesh
rm -f /tmp/mesh-diagnosis.sh
```

---
Next: Day 70 - Autoscaling Failures with HPA and VPA
