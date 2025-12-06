# Day 15: Ingress Path Routing - Order Matters


## THE IDEA:
Create overlapping Ingress path rules and watch requests hit the wrong backend. 
Path precedence, regex vs prefix matching, and trailing slashes create routing 
nightmares.

## THE SETUP:
Deploy two services, create Ingress rules with /api and /api/v2 paths, and 
discover which one catches your traffic. Then explore pathType differences.

## WHAT I LEARNED:
- Prefix pathType matches /api and /api/* (greedy!)
- Exact pathType requires perfect match
- ImplementationSpecific is controller-dependent (nginx ≠ traefik)
- Longest matching prefix wins (usually, but not always)

## WHY IT MATTERS:
Path routing bugs cause:
- 404s for valid endpoints
- Traffic hitting wrong service versions
- Security bypass via path manipulation

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-015

Tomorrow: RBAC lockouts that prevent cluster access.

---

## Killercoda Lab Instructions

### Step 1: Install nginx Ingress controller

```bash
kubectl apply -f <https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml>

# Wait for controller to be ready
kubectl wait --namespace ingress-nginx \\
  --for=condition=ready pod \\
  --selector=app.kubernetes.io/component=controller \\
  --timeout=120s

```

### Step 2: Deploy two backend services

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api-v1
  template:
    metadata:
      labels:
        app: api-v1
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args:
        - "-text=API v1 Response"
        - "-listen=:8080"
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: api-v1-svc
spec:
  selector:
    app: api-v1
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api-v2
  template:
    metadata:
      labels:
        app: api-v2
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args:
        - "-text=API v2 Response"
        - "-listen=:8080"
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: api-v2-svc
spec:
  selector:
    app: api-v2
  ports:
  - port: 80
    targetPort: 8080
EOF

```

### Step 3: Create Ingress with Prefix pathType (greedy matching)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-v1-svc
            port:
              number: 80
      - path: /api/v2
        pathType: Prefix
        backend:
          service:
            name: api-v2-svc
            port:
              number: 80
EOF

```

**Check Ingress status:**

```bash
kubectl get ingress api-ingress
kubectl describe ingress api-ingress

```

### Step 4: Test path routing (unexpected behavior!)

```bash
# Get Ingress IP
INGRESS_IP=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
if [ -z "$INGRESS_IP" ]; then
  INGRESS_IP="localhost"
  kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80 &
  sleep 3
  INGRESS_IP="localhost:8080"
fi

# Test different paths
echo "=== Testing /api ==="
curl -H "Host: api.example.com" http://$INGRESS_IP/api

echo "=== Testing /api/v2 ==="
curl -H "Host: api.example.com" http://$INGRESS_IP/api/v2

echo "=== Testing /api/v2/users ==="
curl -H "Host: api.example.com" http://$INGRESS_IP/api/v2/users

```

**Observation:** `/api` matches FIRST and catches everything!
Even `/api/v2` hits api-v1-svc because Prefix is greedy.

### Step 5: Fix with rule ordering (longest prefix first)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api/v2
        pathType: Prefix
        backend:
          service:
            name: api-v2-svc
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-v1-svc
            port:
              number: 80
EOF

```

**Test again:**

```bash
echo "=== Testing /api/v2 (should hit v2 now) ==="
curl -H "Host: api.example.com" http://$INGRESS_IP/api/v2

echo "=== Testing /api (should hit v1) ==="
curl -H "Host: api.example.com" http://$INGRESS_IP/api

```

Now `/api/v2` routes correctly!

### Step 6: Test Exact pathType

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: exact-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api/users
        pathType: Exact
        backend:
          service:
            name: api-v1-svc
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-v2-svc
            port:
              number: 80
EOF

```

**Test exact matching:**

```bash
echo "=== Testing /api/users (exact match) ==="
curl -H "Host: api.example.com" http://$INGRESS_IP/api/users

echo "=== Testing /api/users/ (trailing slash, no match!) ==="
curl -H "Host: api.example.com" http://$INGRESS_IP/api/users/

echo "=== Testing /api/users/123 (no match, falls to prefix) ==="
curl -H "Host: api.example.com" http://$INGRESS_IP/api/users/123

```

Exact requires perfect match - trailing slash breaks it!

### Step 7: Test ImplementationSpecific (nginx regex)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: regex-ingress
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api/v[0-9]+
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-v2-svc
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-v1-svc
            port:
              number: 80
EOF

```

**Test regex matching:**

```bash
echo "=== Testing /api/v2 (matches regex) ==="
curl -H "Host: api.example.com" http://$INGRESS_IP/api/v2

echo "=== Testing /api/v99 (matches regex) ==="
curl -H "Host: api.example.com" http://$INGRESS_IP/api/v99

echo "=== Testing /api/version (no match, falls to prefix) ==="
curl -H "Host: api.example.com" http://$INGRESS_IP/api/version

```

### Step 8: Path rewriting

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /external/api/v1
        pathType: Prefix
        backend:
          service:
            name: api-v1-svc
            port:
              number: 80
EOF

```

**Test rewriting:**

```bash
echo "=== Testing /external/api/v1 (rewrites to /) ==="
curl -H "Host: api.example.com" http://$INGRESS_IP/external/api/v1

```

Backend receives `/` not `/external/api/v1`.

### Step 9: Multiple Ingress rules (same host)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-1
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /service1
        pathType: Prefix
        backend:
          service:
            name: api-v1-svc
            port:
              number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-2
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /service2
        pathType: Prefix
        backend:
          service:
            name: api-v2-svc
            port:
              number: 80
EOF

```

**Test merged rules:**

```bash
echo "=== Testing /service1 (from ingress-1) ==="
curl -H "Host: api.example.com" http://$INGRESS_IP/service1

echo "=== Testing /service2 (from ingress-2) ==="
curl -H "Host: api.example.com" http://$INGRESS_IP/service2

```

Multiple Ingress objects for same host get merged!

### Step 10: Debugging Ingress routing

```bash
# Check nginx config
NGINX_POD=$(kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n ingress-nginx $NGINX_POD -- cat /etc/nginx/nginx.conf | grep -A 20 "server_name api.example.com"

```

View actual nginx routing rules generated.

**Check Ingress controller logs:**

```bash
kubectl logs -n ingress-nginx $NGINX_POD --tail=50

```

### Key Observations

✅ **Prefix pathType** - greedy, matches path + subpaths
✅ **Rule ordering matters** - longest prefix should come first
✅ **Exact is brittle** - trailing slashes, case sensitivity
✅ **ImplementationSpecific** - behavior varies by controller
✅ **Multiple Ingress merge** - same host rules combine

### Production Patterns

**API versioning (correct order):**

```yaml
paths:
- path: /api/v3
  pathType: Prefix
  backend:
    service:
      name: api-v3
- path: /api/v2
  pathType: Prefix
  backend:
    service:
      name: api-v2
- path: /api
  pathType: Prefix
  backend:
    service:
      name: api-v1

```

**Microservices routing:**

```yaml
paths:
- path: /users
  pathType: Prefix
  backend:
    service:
      name: user-service
- path: /orders
  pathType: Prefix
  backend:
    service:
      name: order-service
- path: /
  pathType: Prefix
  backend:
    service:
      name: frontend

```

**Regex for dynamic routes:**

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  rules:
  - http:
      paths:
      - path: /api/v[0-9]+/.*
        pathType: ImplementationSpecific

```

### Cleanup

```bash
pkill -f "port-forward.*ingress-nginx" 2>/dev/null
kubectl delete ingress --all
kubectl delete deployment api-v1 api-v2
kubectl delete service api-v1-svc api-v2-svc

```

---

Next: Day 16 - RBAC Permission Denied Debugging