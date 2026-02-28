## Day 84: Ingress Controller Failures - Traffic Can't Route

### Email Copy

**Subject:** Day 84: Ingress Controller Failures - When 404s Come from Nowhere

THE IDEA:
Deploy Ingress resources and watch traffic fail. 404 errors on valid paths, TLS
certificates don't work, backends unreachable, path-based routing broken.

THE SETUP:
Deploy Ingress without controller, use wrong annotations, break path rewrites,
misconfigure TLS, and discover what fails when HTTP routing breaks.

WHAT I LEARNED:
- Ingress resource needs Ingress controller
- Different controllers use different annotations
- Path type matters (Prefix vs Exact)
- TLS secrets must be in same namespace
- Backend service must exist and have endpoints
- IngressClass selects which controller handles it

WHY IT MATTERS:
Ingress failures cause:
- All external traffic returns 404
- SSL/TLS errors for end users
- Services unreachable from outside
- Load balancing doesn't work
- Path-based routing fails

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-084

Tomorrow: Job and CronJob scheduling disasters.

---
Chad


---

## Killercoda Lab Instructions

## Step 1: Check for Ingress controller

```bash
# Check if Ingress controller exists
kubectl get pods -A | grep ingress
```

```bash
# Check IngressClass
kubectl get ingressclass

echo ""
echo "Common Ingress controllers:"
echo "- nginx-ingress-controller"
echo "- traefik"
echo "- haproxy-ingress"
echo "- AWS ALB Ingress Controller"
echo "- GCE Ingress Controller"
```

## Step 2: Deploy backend service

```bash
# Deploy application to route to
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args:
        - "-text=Hello from web-app"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 5678
EOF

kubectl wait --for=condition=Ready pod -l app=web --timeout=60s
```

## Step 3: Create Ingress without controller

```bash
# Ingress resource (no controller to implement it!)
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: broken-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
EOF

sleep 10
```

```bash
# Check Ingress status
kubectl get ingress broken-ingress

echo ""
echo "Ingress has no ADDRESS (no controller to provision it)"
kubectl describe ingress broken-ingress | grep -A 5 "Events:"
```

## Step 4: Test wrong IngressClass

```bash
# Ingress with non-existent class
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wrong-class-ingress
spec:
  ingressClassName: nonexistent-class
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
EOF

sleep 10

kubectl get ingress wrong-class-ingress

echo ""
echo "Ingress ignored by controllers (IngressClass doesn't exist)"
```

## Step 5: Test missing backend service

```bash
# Ingress pointing to non-existent service
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: missing-backend-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: nonexistent-service  # Doesn't exist!
            port:
              number: 80
EOF

sleep 10

kubectl describe ingress missing-backend-ingress | grep -i "error\|warning"

echo ""
echo "Backend service doesn't exist - traffic will 404 or 503"
```

## Step 6: Test path type confusion

```bash
# Deploy additional backend
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args:
        - "-text=API Response"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 5678
EOF

kubectl wait --for=condition=Ready pod -l app=api --timeout=60s
```

```bash
# Ingress with different path types
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-types-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Exact  # Only matches /api exactly
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix  # Matches / and everything under it
        backend:
          service:
            name: web-service
            port:
              number: 80
EOF

echo "Path type differences:"
echo "Exact: /api matches, /api/ doesn't match"
echo "Prefix: /api matches /api, /api/v1, /api/users"
echo "ImplementationSpecific: Controller-dependent"
```

## Step 7: Test TLS certificate issues

```bash
# Create dummy TLS secret
cat > /tmp/fake-tls.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: example-tls
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURBRENDQWVpZ0F3SUJBZ0lKQUtHMDZycldOdGg1TUEwR0NTcUdTSWIzRFFFQkN3VUFNQmt4RnpBVgpCZ05WQkFNTURuZDNkeTVsZUdGdGNHeGxMbU52YlRBZUZ3MHlNekExTURNd01EQXdNREJhRncwek16QTAKTVRNZ01EQXdNREJhTUJreEZ6QVZCZ05WQkFNTURuZDNkeTVsZUdGdGNHeGxMbU52YlRDQ0FTSXdEUVlKCktvWklodmNOQVFFQkJRQURnZ0VQQURDQ0FRb0NnZ0VCQUt3SzBvL3pFdlhEb1d4eEd2amRvQXYvZis4NQp4eGZQMEVaVzN0dDVGdGc0TjFUSjhZSkVGREVzZ1E9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  tls.key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUV2UUlCQURBTkJna3Foa2lHOXcwQkFRRUZBQVNDQktjd2dnU2pBZ0VBQWpFQU1BQT0KLS0tLS1FTkQgUFJJVkFURSBLRVktLS0tLQo=
EOF

kubectl apply -f /tmp/fake-tls.yaml
```

```bash
# Ingress with TLS
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - example.com
    secretName: example-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
EOF

echo "TLS configuration:"
echo "- Secret must be type kubernetes.io/tls"
echo "- Must have tls.crt and tls.key"
echo "- Must be in same namespace as Ingress"
echo "- Certificate must match hostname"
```

## Step 8: Test TLS secret in wrong namespace

```bash
# Create namespace
kubectl create namespace other-ns
```

```bash
# Ingress references secret in different namespace
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cross-namespace-tls
  namespace: other-ns
spec:
  tls:
  - hosts:
    - example.com
    secretName: example-tls  # In default namespace!
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
EOF

sleep 10

kubectl describe ingress cross-namespace-tls -n other-ns | grep -i "error\|warning"

echo ""
echo "TLS secret not found - must be in same namespace as Ingress"
```

## Step 9: Test annotation incompatibility

```bash
# Ingress with nginx-specific annotations
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-annotations
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    # These only work with nginx Ingress controller!
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /app
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
EOF

echo "Annotation incompatibility:"
echo "- nginx.ingress.kubernetes.io/* only for nginx"
echo "- traefik.ingress.kubernetes.io/* only for Traefik"
echo "- alb.ingress.kubernetes.io/* only for AWS ALB"
echo "- Wrong annotations = ignored or errors"
```

## Step 10: Test path rewrite issues

```bash
# Ingress with broken rewrite
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /\$2
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /app(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: web-service
            port:
              number: 80
EOF

echo "Path rewrite issues:"
echo "- Request: /app/users"
echo "- Rewrite to: /users (if working)"
echo "- Common problems:"
echo "  - Wrong regex in path"
echo "  - Wrong capture group in rewrite-target"
echo "  - PathType must be ImplementationSpecific"
```

## Step 11: Test backend without endpoints

```bash
# Service with no matching pods
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: no-endpoints-service
spec:
  selector:
    app: nonexistent  # No pods match!
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: no-endpoints-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /broken
        pathType: Prefix
        backend:
          service:
            name: no-endpoints-service
            port:
              number: 80
EOF

sleep 10
```

```bash
# Check service endpoints
kubectl get endpoints no-endpoints-service

echo ""
echo "Service has no endpoints - traffic will 503"
```

## Step 12: Test conflicting Ingress rules

```bash
# Multiple Ingresses for same host
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: conflict-ingress-1
spec:
  rules:
  - host: conflict.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: conflict-ingress-2
spec:
  rules:
  - host: conflict.example.com  # Same host!
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
EOF

echo "Conflicting Ingress rules:"
echo "- Multiple Ingresses for same host"
echo "- Behavior depends on controller"
echo "- May merge or conflict"
echo "- Better to use one Ingress with multiple paths"
```

## Step 13: Test rate limiting

```bash
# Ingress with rate limiting
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rate-limited-ingress
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "1"
    nginx.ingress.kubernetes.io/limit-connections: "1"
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
EOF

echo "Rate limiting annotations:"
echo "- limit-rps: requests per second"
echo "- limit-connections: concurrent connections"
echo "- Exceeding limits = 503 responses"
echo "- Applied per IP address"
```

## Step 14: Test default backend

```bash
# Ingress with default backend
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: default-backend-ingress
spec:
  defaultBackend:
    service:
      name: web-service
      port:
        number: 80
  rules:
  - host: specific.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
EOF

echo "Default backend behavior:"
echo "- Catches all unmatched requests"
echo "- No host specified = uses default"
echo "- No path match = uses default"
echo "- Useful for custom 404 pages"
```

## Step 15: Ingress troubleshooting guide

```bash
cat > /tmp/ingress-troubleshooting.md << 'EOF'
# Ingress Troubleshooting Guide

## Diagnosis Steps

### 1. Check Ingress Controller
kubectl get pods -A | grep ingress
kubectl logs -n <namespace> <ingress-controller-pod>

### 2. Check IngressClass
kubectl get ingressclass
# Check default class
kubectl get ingressclass -o json | jq '.items[] | select(.metadata.annotations."ingressclass.kubernetes.io/is-default-class"=="true")'

### 3. Verify Ingress Resource
kubectl get ingress <name> -o yaml
kubectl describe ingress <name>
# Look for ADDRESS field - empty = controller not processing

### 4. Test Backend Service
kubectl get svc <service-name>
kubectl get endpoints <service-name>
kubectl run test --rm -i --image=busybox -- wget -O- <service-name>

### 5. Test TLS Configuration
kubectl get secret <tls-secret>
kubectl get secret <tls-secret> -o jsonpath='{.type}'

## Common Problems

### 404 Not Found
- Wrong path in Ingress
- PathType mismatch (Exact vs Prefix)
- Service name typo
- Host header doesn't match

### 503 Service Unavailable
- Service has no endpoints
- All pods are not ready
- Backend pods crashed
- Rate limit exceeded

### TLS/SSL Errors
- Certificate doesn't match hostname
- Certificate expired
- Secret not found or wrong type
- Secret in wrong namespace

### Ingress Not Getting IP
- Ingress controller not running
- Wrong IngressClass specified
- Cloud provider issue

## Best Practices
1. Use a single Ingress per host with multiple paths
2. Always specify ingressClassName explicitly
3. Use cert-manager for TLS certificate management
4. Monitor backend endpoint health
5. Set timeouts appropriate to your application
EOF

cat /tmp/ingress-troubleshooting.md
```

## Key Observations

✅ **Controller required** - Ingress is just config
✅ **IngressClass** - selects which controller processes it
✅ **Path types** - Exact vs Prefix behavior differs
✅ **TLS secrets** - must be in same namespace
✅ **Backend endpoints** - service needs ready pods
✅ **Annotations** - controller-specific

## Production Patterns

**Complete Ingress setup:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "30"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

## Cleanup

```bash
kubectl delete ingress --all
kubectl delete deployment web-app api-app
kubectl delete service web-service api-service no-endpoints-service
kubectl delete secret example-tls
kubectl delete namespace other-ns 2>/dev/null
rm -f /tmp/*.yaml /tmp/*.md
```

---
Next: Day 85 - Job and CronJob Scheduling Disasters
