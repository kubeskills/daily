# Day 11: Service DNS Propagation - The Timing Gap

## THE IDEA:
Deploy a service and immediately try to connect. Watch it fail because DNS
hasn't propagated yet. Explore TTL settings, headless services, and why
connection pooling makes this worse.

## THE SETUP:
Create a deployment, then a service, and race to connect before DNS is ready.
Test CoreDNS caching behavior and endpoint propagation delays.

## WHAT I LEARNED:
- Service creation → DNS availability: 0-5 seconds typically
- Endpoint slice updates can lag behind pod readiness
- Connection pools cache resolved IPs (stale after pod replacement)
- Headless services give you direct pod IPs (no caching issues)

## WHY IT MATTERS:
DNS timing issues cause:
- Intermittent "connection refused" during deployments
- Stale connections after pod replacement
- Init containers failing to find services

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-011

---

## Killercoda Lab Instructions

### Step 1: Check CoreDNS configuration

```bash
kubectl get configmap coredns -n kube-system -o yaml

```

Look for the `ttl` setting (default is usually 30 seconds).

### Step 2: Create backend deployment and service simultaneously

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: web
        image: hashicorp/http-echo
        args: ["-text=Backend response", "-listen=:8080"]
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 2
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
EOF

```

### Step 3: Immediately try to resolve DNS

```bash
# Race condition test - run immediately after apply
kubectl run dns-test --rm -it --restart=Never --image=busybox -- nslookup backend-svc

```

If pods aren't ready yet, you might get NXDOMAIN or empty response.

### Step 4: Watch endpoint propagation

```bash
kubectl get endpoints backend-svc -w

```

Notice the delay between deployment creation and endpoints appearing.
Endpoints only populate after pods pass readiness checks.

### Step 5: Test DNS resolution timing

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: dns-timing-test
spec:
  containers:
  - name: test
    image: busybox
    command: ['sh', '-c', '
      echo "Starting DNS resolution test...";
      for i in $(seq 1 20); do
        START=$(date +%s%N);
        nslookup backend-svc > /dev/null 2>&1;
        END=$(date +%s%N);
        DURATION=$(( ($END - $START) / 1000000 ));
        echo "Query $i: ${DURATION}ms";
        sleep 1;
      done
    ']
EOF

```

**Check timing:**

```bash
kubectl logs dns-timing-test -f

```

First query might be slower (cache miss), subsequent queries faster.

### Step 6: Test headless service (direct pod IPs)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: backend-headless
spec:
  clusterIP: None  # Headless!
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 8080
EOF

```

**Compare DNS responses:**

```bash
# Regular service (single ClusterIP)
kubectl run tmp1 --rm -it --restart=Never --image=busybox -- nslookup backend-svc

# Headless service (multiple pod IPs)
kubectl run tmp2 --rm -it --restart=Never --image=busybox -- nslookup backend-headless

```

Headless returns individual pod IPs, regular returns single ClusterIP.

### Step 7: Simulate stale DNS during rolling update

```bash
# Terminal 1: Watch endpoints
kubectl get endpoints backend-svc -w

# Terminal 2: Start continuous requests
kubectl run client --rm -it --restart=Never --image=curlimages/curl -- sh -c '
while true; do
  RESP=$(curl -s -o /dev/null -w "%{http_code}" --connect-timeout 2 <http://backend-svc>)
  echo "$(date +%H:%M:%S) - HTTP $RESP"
  sleep 0.5
done
'

# Terminal 3: Trigger rolling update
kubectl set image deployment/backend web=hashicorp/http-echo:latest

```

Watch for brief connection failures during the rollout.

### Step 8: Test init container DNS dependency

```bash
kubectl delete service backend-svc

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: init-dns-test
spec:
  initContainers:
  - name: wait-for-backend
    image: busybox
    command: ['sh', '-c', '
      echo "Waiting for backend-svc DNS...";
      for i in $(seq 1 30); do
        if nslookup backend-svc; then
          echo "Service found!";
          exit 0;
        fi;
        echo "Attempt $i failed, waiting...";
        sleep 2;
      done;
      echo "Timeout waiting for service";
      exit 1
    ']
  containers:
  - name: app
    image: nginx
EOF

```

**Watch it wait:**

```bash
kubectl get pod init-dns-test -w
kubectl logs init-dns-test -c wait-for-backend -f

```

Init container retries until service DNS is available.

### Step 9: Create the service and watch init complete

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
EOF

```

Watch the init container succeed and main container start!

### Step 10: DNS policy options

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: dns-policy-test
spec:
  dnsPolicy: ClusterFirst  # Default: use cluster DNS
  # dnsPolicy: Default      # Use node's DNS
  # dnsPolicy: None         # Custom DNS config only
  dnsConfig:
    options:
    - name: ndots
      value: "2"     # Reduce search domain lookups
    - name: timeout
      value: "2"     # 2 second timeout
    - name: attempts
      value: "3"     # 3 retry attempts
  containers:
  - name: test
    image: busybox
    command: ['sleep', '3600']
EOF

```

**Check resolv.conf:**

```bash
kubectl exec dns-policy-test -- cat /etc/resolv.conf

```

Key Observations

✅ **DNS propagation** - typically 0-5 seconds after service creation
✅ **Endpoints lag** - wait for pod readiness before endpoints populate
✅ **Headless services** - return pod IPs directly, bypass ClusterIP caching
✅ **Connection pools** - cache IPs, need refresh logic for pod replacements
✅ **ndots setting** - affects how many dots required before absolute lookup

Production Patterns

**Reduce DNS lookup latency:**

```yaml
dnsConfig:
  options:
  - name: ndots
    value: "2"  # Default is 5, which causes many unnecessary lookups

```

**Mesh/sidecar for connection management:**

- Istio/Linkerd handle connection pooling and retry
- Automatic endpoint refresh on pod changes

**ExternalName for external services:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.example.com

```

Cleanup

```bash
kubectl delete pod dns-timing-test init-dns-test dns-policy-test client 2>/dev/null
kubectl delete service backend-svc backend-headless 2>/dev/null
kubectl delete deployment backend 2>/dev/null

```

---

Next: Day 12 - HPA Thrashing