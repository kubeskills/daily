## Day 58: Ingress Failures - Traffic Not Routing

### Email Copy

**Subject:** Day 58: Ingress Failures - When Traffic Disappears

THE IDEA:
Deploy Ingress resources and watch traffic fail to reach services due to missing
IngressClass, backend misconfigurations, and TLS certificate issues.

THE SETUP:
Create Ingress with wrong service names, missing IngressClass, conflicting paths,
and broken TLS. Debug why external traffic can't reach your apps.

WHAT I LEARNED:
- IngressClass required in K8s 1.18+ (no default)
- Service port must match Ingress backend port
- Path matching: Prefix vs Exact
- TLS secret must be in same namespace as Ingress
- Multiple Ingresses can conflict on same host

WHY IT MATTERS:
Ingress issues cause:
- Complete service unavailability from outside cluster
- 404 errors despite services being healthy
- TLS certificate errors blocking all traffic

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-058

Tomorrow: Helm chart deployment failures (Week 9 finale!).

---
Chad



---

## Killercoda Lab Instructions


## Step 1: Install NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

# Wait for ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

## Step 2: Deploy test backend services

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo
      version: v1
  template:
    metadata:
      labels:
        app: demo
        version: v1
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-text=Version 1"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: app-v1-svc
spec:
  selector:
    app: demo
    version: v1
  ports:
  - port: 80
    targetPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo
      version: v2
  template:
    metadata:
      labels:
        app: demo
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
  name: app-v2-svc
spec:
  selector:
    app: demo
    version: v2
  ports:
  - port: 80
    targetPort: 5678
EOF

kubectl wait --for=condition=Ready pods -l app=demo --timeout=60s
```

## Step 3: Create Ingress without IngressClass (fails in K8s 1.18+)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: broken-ingress
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v1-svc
            port:
              number: 80
EOF

# Check status
kubectl get ingress broken-ingress
kubectl describe ingress broken-ingress
```

No ADDRESS assigned - missing IngressClass!

## Step 4: Fix with IngressClass

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: working-ingress
spec:
  ingressClassName: nginx  # Required!
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v1-svc
            port:
              number: 80
EOF

# Check status
kubectl get ingress working-ingress
```

Now has ADDRESS!

## Step 5: Test wrong service name

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wrong-service
spec:
  ingressClassName: nginx
  rules:
  - host: wrong.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nonexistent-service  # Doesn't exist!
            port:
              number: 80
EOF

# Check Ingress events
kubectl describe ingress wrong-service | grep -A 10 "Events:"
```

## Step 6: Test wrong port number

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wrong-port
spec:
  ingressClassName: nginx
  rules:
  - host: port.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v1-svc
            port:
              number: 8080  # Wrong! Service is on port 80
EOF
```

**Test (returns 503):**
```bash
INGRESS_IP=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || echo "localhost")
curl -H "Host: port.example.com" http://$INGRESS_IP/ 2>&1 || echo "503 Service Unavailable"
```

## Step 7: Test path type differences

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-types
spec:
  ingressClassName: nginx
  rules:
  - host: paths.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix  # Matches /v1, /v1/, /v1/anything
        backend:
          service:
            name: app-v1-svc
            port:
              number: 80
      - path: /v2/exact
        pathType: Exact  # Only matches /v2/exact
        backend:
          service:
            name: app-v2-svc
            port:
              number: 80
EOF

# Test Prefix matching
curl -H "Host: paths.example.com" http://$INGRESS_IP/v1/test

# Test Exact matching
curl -H "Host: paths.example.com" http://$INGRESS_IP/v2/exact
curl -H "Host: paths.example.com" http://$INGRESS_IP/v2/exact/more  # Should 404
```

## Step 8: Test conflicting Ingress rules

```bash
# Create two Ingresses with same host
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: conflict-a
spec:
  ingressClassName: nginx
  rules:
  - host: conflict.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v1-svc
            port:
              number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: conflict-b
spec:
  ingressClassName: nginx
  rules:
  - host: conflict.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v2-svc
            port:
              number: 80
EOF

# Check which one wins
kubectl describe ingress conflict-a conflict-b | grep -A 5 "Rules:"
```

Last applied wins (non-deterministic)!

## Step 9: Test TLS with missing secret

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-broken
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - tls.example.com
    secretName: nonexistent-tls-secret  # Doesn't exist!
  rules:
  - host: tls.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v1-svc
            port:
              number: 80
EOF

# Check status
kubectl describe ingress tls-broken | grep -A 10 "Events:"
```

## Step 10: Create valid TLS secret

```bash
# Generate self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/tls.key -out /tmp/tls.crt \
  -subj "/CN=tls.example.com/O=tls.example.com"

# Create secret
kubectl create secret tls tls-secret \
  --cert=/tmp/tls.crt \
  --key=/tmp/tls.key

# Create Ingress with TLS
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-working
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - tls.example.com
    secretName: tls-secret
  rules:
  - host: tls.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v1-svc
            port:
              number: 80
EOF

# Test HTTPS
curl -k -H "Host: tls.example.com" https://$INGRESS_IP/
```

<!-- Content was truncated. Please add the remaining steps. -->
