## Day 33: Service Mesh - Traffic Split Gone Wrong

## THE IDEA:
Install Istio, configure traffic splitting between v1 and v2 of a service, and 
introduce routing bugs that send 100% traffic to the wrong version or cause 
503 errors.

## THE SETUP:
Deploy Istio, create VirtualService with weighted routing, test canary deployment, 
and break it with conflicting rules or missing destination subsets.

## WHAT I LEARNED:
- VirtualService defines routing rules
- DestinationRule defines service subsets (versions)
- Weighted routing must add up to 100
- Missing subset causes 503 Service Unavailable
- Rule precedence matters (first match wins)

## WHY IT MATTERS:
Service mesh routing issues cause:
- All traffic to canary version (oops!)
- 503 errors for valid requests
- Traffic blackholes from misconfigured rules

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-033

Tomorrow: Multi-tenancy namespace isolation failures (Week 5 finale!).

---

# Killercoda Lab Instructions


## Step 1: Install Istio

```bash
# Download Istio
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.20.0 sh -
cd istio-1.20.0
export PATH=$PWD/bin:$PATH

# Install Istio
istioctl install --set profile=demo -y

# Label namespace for injection
kubectl label namespace default istio-injection=enabled
```

## Step 2: Verify Istio installation

```bash
kubectl get pods -n istio-system
kubectl get svc -n istio-system
```

## Step 3: Deploy application v1

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
      version: v1
  template:
    metadata:
      labels:
        app: webapp
        version: v1
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args:
        - "-text=Version 1"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 5678
EOF
```

**Wait for pods:**
```bash
kubectl get pods -l app=webapp
```

## Step 4: Deploy application v2

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-v2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
      version: v2
  template:
    metadata:
      labels:
        app: webapp
        version: v2
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args:
        - "-text=Version 2 - NEW!"
        ports:
        - containerPort: 5678
EOF
```

## Step 5: Test without routing rules (random distribution)

```bash
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- sh -c '
for i in $(seq 1 10); do
  curl -s http://webapp
  sleep 0.5
done
'
```

Random mix of v1 and v2!

## Step 6: Create DestinationRule (define subsets)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: webapp
spec:
  host: webapp
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
EOF
```

## Step 7: Create VirtualService (90/10 split)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: webapp
spec:
  hosts:
  - webapp
  http:
  - route:
    - destination:
        host: webapp
        subset: v1
      weight: 90
    - destination:
        host: webapp
        subset: v2
      weight: 10
EOF
```

**Test traffic split:**
```bash
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- sh -c '
for i in $(seq 1 20); do
  curl -s http://webapp
done
' | grep -c "Version 1"
```

~18 requests to v1, ~2 to v2

## Step 8: Break routing - weights don't add to 100

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: webapp
spec:
  hosts:
  - webapp
  http:
  - route:
    - destination:
        host: webapp
        subset: v1
      weight: 90
    - destination:
        host: webapp
        subset: v2
      weight: 20  # Total = 110!
EOF
```

**Test:**
```bash
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- curl -s http://webapp
```

Still works! Istio normalizes weights.

## Step 9: Break routing - missing subset

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: webapp
spec:
  hosts:
  - webapp
  http:
  - route:
    - destination:
        host: webapp
        subset: v3  # Doesn't exist!
      weight: 100
EOF
```

**Test:**
```bash
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- curl -v http://webapp 2>&1 | grep -i "503\|error"
```

503 Service Unavailable!

## Step 10: Check Istio proxy logs

```bash
POD=$(kubectl get pods -l app=webapp,version=v1 -o jsonpath='{.items[0].metadata.name}')
kubectl logs $POD -c istio-proxy | tail -20
```

Shows routing errors.

## Step 11: Fix and test header-based routing

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: webapp
spec:
  hosts:
  - webapp
  http:
  - match:
    - headers:
        x-version:
          exact: v2
    route:
    - destination:
        host: webapp
        subset: v2
  - route:
    - destination:
        host: webapp
        subset: v1
EOF
```

**Test default (v1):**
```bash
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- curl -s http://webapp
```

**Test with header (v2):**
```bash
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- curl -H "x-version: v2" -s http://webapp
```

Header routing works!

## Step 12: Test conflicting rules (first match wins)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: webapp
spec:
  hosts:
  - webapp
  http:
  - match:
    - uri:
        prefix: /api
    route:
    - destination:
        host: webapp
        subset: v2
  - match:
    - uri:
        prefix: /  # Catches everything!
    route:
    - destination:
        host: webapp
        subset: v1
EOF
```

Order matters! First matching rule wins.

## Step 13: Test retry configuration

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: webapp
spec:
  hosts:
  - webapp
  http:
  - route:
    - destination:
        host: webapp
        subset: v1
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx,reset,connect-failure
EOF
```

## Step 14: Test timeout configuration

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: webapp
spec:
  hosts:
  - webapp
  http:
  - route:
    - destination:
        host: webapp
        subset: v1
    timeout: 1s  # Very aggressive!
EOF
```

**Deploy slow app:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-v3-slow
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
      version: v3
  template:
    metadata:
      labels:
        app: webapp
        version: v3
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args:
        - "-text=Slow version"
        - "-listen=:5678"
        ports:
        - containerPort: 5678
        lifecycle:
          postStart:
            exec:
              command: ['sh', '-c', 'sleep 5']
EOF
```

## Step 15: Check VirtualService status

```bash
kubectl get virtualservice webapp -o yaml
istioctl analyze
```

Shows configuration warnings!

## Key Observations

✅ **VirtualService** - defines routing rules
✅ **DestinationRule** - defines service subsets
✅ **Weight-based routing** - traffic splitting for canary
✅ **Header-based routing** - route by request headers
✅ **Missing subset** - causes 503 errors
✅ **Rule order** - first match wins

## Production Patterns

**Canary deployment (10% traffic):**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: webapp-canary
spec:
  hosts:
  - webapp
  http:
  - route:
    - destination:
        host: webapp
        subset: stable
      weight: 90
    - destination:
        host: webapp
        subset: canary
      weight: 10
```

**Blue/green deployment:**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: webapp-bluegreen
spec:
  hosts:
  - webapp
  http:
  - match:
    - headers:
        x-env:
          exact: green
    route:
    - destination:
        host: webapp
        subset: green
  - route:
    - destination:
        host: webapp
        subset: blue
```

**Fault injection for testing:**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: webapp-fault
spec:
  hosts:
  - webapp
  http:
  - fault:
      delay:
        percentage:
          value: 10
        fixedDelay: 5s
      abort:
        percentage:
          value: 5
        httpStatus: 500
    route:
    - destination:
        host: webapp
```

## Cleanup

```bash
kubectl delete virtualservice webapp
kubectl delete destinationrule webapp
kubectl delete deployment webapp-v1 webapp-v2 webapp-v3-slow
kubectl delete service webapp
istioctl uninstall --purge -y
kubectl delete namespace istio-system
kubectl label namespace default istio-injection-
cd ..
rm -rf istio-1.20.0
```

---
Next: Day 34 - Multi-Tenancy Namespace Isolation Failures