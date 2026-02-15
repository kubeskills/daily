## Day 46: Probe Tuning - The Goldilocks Problem

## THE IDEA:
Deploy apps with aggressive probes (restart loops), lenient probes (serve traffic while unhealthy), and balanced probes. Learn to tune for your app's behavior.

## THE SETUP:
Create probes with different timeouts, periods, and thresholds. Trigger failures and observe whether the pod restarts too quickly or stays unhealthy too long.

## WHAT I LEARNED:
- Liveness kills pod, readiness removes from service
- initialDelaySeconds prevents premature checks
- failureThreshold determines retry patience
- Startup probe delays liveness during slow startup
- Probe overhead can impact performance

## WHY IT MATTERS:
Probe misconfiguration causes:
- Restart loops from impatient probes
- Serving traffic to unhealthy pods
- Slow failure detection and recovery

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-046

Tomorrow: StatefulSet scaling and persistent volume orphans (Week 7 finale!).


---

# Killercoda Lab Instructions


## Step 1: Deploy app with aggressive liveness probe (restart loop)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: aggressive-probe
spec:
  containers:
  - name: app
    image: hashicorp/http-echo
    args: ["-text=Hello", "-listen=:8080"]
    ports:
    - containerPort: 8080
    livenessProbe:
      httpGet:
        path: /
        port: 8080
      initialDelaySeconds: 1
      periodSeconds: 2
      timeoutSeconds: 1
      failureThreshold: 1  # Restart after 1 failure!
EOF
```

**Watch restart loop:**
```bash
kubectl get pod aggressive-probe -w
```

Even brief network hiccup causes restart!

## Step 2: Deploy app with slow startup (liveness kills it)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: slow-startup
spec:
  containers:
  - name: app
    image: hashicorp/http-echo
    args: ["-text=Slow starter", "-listen=:8080"]
    ports:
    - containerPort: 8080
    lifecycle:
      postStart:
        exec:
          command: ['sh', '-c', 'sleep 30']  # Simulates slow startup
    livenessProbe:
      httpGet:
        path: /
        port: 8080
      initialDelaySeconds: 5  # Too short!
      periodSeconds: 5
      failureThreshold: 3
EOF
```

**Check status:**
```bash
kubectl get pod slow-startup -w
```

Liveness probe fails before app is ready → restart!

## Step 3: Fix with startupProbe

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: startup-probe-fixed
spec:
  containers:
  - name: app
    image: hashicorp/http-echo
    args: ["-text=Fixed", "-listen=:8080"]
    ports:
    - containerPort: 8080
    lifecycle:
      postStart:
        exec:
          command: ['sh', '-c', 'sleep 30']
    startupProbe:
      httpGet:
        path: /
        port: 8080
      periodSeconds: 5
      failureThreshold: 10  # 50 seconds total
    livenessProbe:
      httpGet:
        path: /
        port: 8080
      periodSeconds: 10
      failureThreshold: 3
EOF
```

**Verify:**
```bash
kubectl get pod startup-probe-fixed -w
```

Startup probe allows time, then liveness takes over!

## Step 4: Test readiness probe (traffic routing)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness-test
spec:
  replicas: 3
  selector:
    matchLabels:
      app: readiness
  template:
    metadata:
      labels:
        app: readiness
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-text=Pod \$HOSTNAME", "-listen=:8080"]
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 3
          failureThreshold: 2
---
apiVersion: v1
kind: Service
metadata:
  name: readiness-svc
spec:
  selector:
    app: readiness
  ports:
  - port: 80
    targetPort: 8080
EOF
```

**Check endpoints:**
```bash
kubectl get endpoints readiness-svc
```

Only ready pods in endpoints!

## Step 5: Simulate readiness failure

```bash
# Kill one pod's readiness
POD=$(kubectl get pods -l app=readiness -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- killall -9 http-echo

# Check endpoints
kubectl get endpoints readiness-svc -w
```

Failed pod removed from service!

## Step 6: Test exec probe

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: exec-probe
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'touch /tmp/healthy; sleep 3600']
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 10
EOF
```

**Trigger failure:**
```bash
kubectl exec exec-probe -- rm /tmp/healthy

# Watch restart
kubectl get pod exec-probe -w
```

## Step 7: Test TCP probe

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: tcp-probe
spec:
  containers:
  - name: app
    image: nginx
    ports:
    - containerPort: 80
    livenessProbe:
      tcpSocket:
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
    readinessProbe:
      tcpSocket:
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 5
EOF
```

TCP probes check if port is open!

## Step 8: Test gRPC probe (K8s 1.24+)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: grpc-probe
spec:
  containers:
  - name: app
    image: hashicorp/http-echo
    args: ["-listen=:8080"]
    ports:
    - containerPort: 8080
    livenessProbe:
      grpc:
        port: 8080
      initialDelaySeconds: 5
EOF
```

## Step 9: Tune probe timing

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: tuned-probes
spec:
  containers:
  - name: app
    image: nginx
    ports:
    - containerPort: 80
    startupProbe:
      httpGet:
        path: /
        port: 80
      periodSeconds: 5
      failureThreshold: 30  # 150s max startup time
    livenessProbe:
      httpGet:
        path: /
        port: 80
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3  # 30s to recover
    readinessProbe:
      httpGet:
        path: /
        port: 80
      periodSeconds: 2
      timeoutSeconds: 1
      failureThreshold: 1  # Immediate removal
      successThreshold: 1
EOF
```

## Step 10: Test probe overhead

```bash
# Deploy with high-frequency probes
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: probe-overhead
spec:
  replicas: 100  # Many pods
  selector:
    matchLabels:
      app: overhead
  template:
    metadata:
      labels:
        app: overhead
    spec:
      containers:
      - name: app
        image: nginx
        readinessProbe:
          httpGet:
            path: /
            port: 80
          periodSeconds: 1  # Every second!
        livenessProbe:
          httpGet:
            path: /
            port: 80
          periodSeconds: 1
EOF
```

**Check node CPU:**
```bash
kubectl top nodes
```

Probe overhead visible with many pods!

## Step 11: Test custom health endpoint

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: custom-health
spec:
  containers:
  - name: app
    image: hashicorp/http-echo
    args: ["-text=Healthy", "-listen=:8080"]
    ports:
    - containerPort: 8080
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: X-Custom-Header
          value: ProbeRequest
      initialDelaySeconds: 5
      periodSeconds: 10
EOF
```

## Step 12: Test probe failure during rolling update

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-update
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rolling
  template:
    metadata:
      labels:
        app: rolling
        version: v1
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-text=v1", "-listen=:8080"]
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          periodSeconds: 2
          failureThreshold: 3
EOF
```

**Update to failing version:**
```bash
kubectl set image deployment/rolling-update app=hashicorp/http-echo:nonexistent

# Watch rollout get stuck
kubectl rollout status deployment/rolling-update --timeout=30s
```

Readiness probe prevents bad rollout!

## Step 13: Check probe metrics

```bash
# Describe pod to see probe results
kubectl describe pod tuned-probes | grep -A 10 "Liveness\|Readiness\|Startup"

# Check events for probe failures
kubectl get events --field-selector involvedObject.name=tuned-probes --sort-by='.lastTimestamp'
```

## Step 14: Test terminationGracePeriodSeconds interaction

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: graceful-shutdown
spec:
  terminationGracePeriodSeconds: 30
  containers:
  - name: app
    image: nginx
    lifecycle:
      preStop:
        exec:
          command: ['sh', '-c', 'sleep 20']
    livenessProbe:
      httpGet:
        path: /
        port: 80
      periodSeconds: 5
EOF
```

**Delete and watch:**
```bash
kubectl delete pod graceful-shutdown
kubectl get pod graceful-shutdown -w
```

## Step 15: Best practices summary

```yaml
# Production-ready probe configuration
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      periodSeconds: 10
      failureThreshold: 30  # 5 minutes max startup
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 0  # startupProbe handles this
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3  # 30 seconds to recover
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 5
      timeoutSeconds: 2
      failureThreshold: 2
      successThreshold: 1
```

## Key Observations

✅ **Liveness** - restarts pod on failure
✅ **Readiness** - removes from service on failure
✅ **Startup** - delays liveness during slow startup
✅ **Timing tuning** - balance sensitivity vs patience
✅ **Probe overhead** - high frequency impacts performance
✅ **Rolling updates** - readiness prevents bad deployments

## Production Patterns

**Fast-starting web app:**
```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 2
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
  failureThreshold: 3
```

**Slow-starting Java app:**
```yaml
startupProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  periodSeconds: 10
  failureThreshold: 30  # 5 minutes
livenessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  periodSeconds: 20
  failureThreshold: 3
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  periodSeconds: 10
  failureThreshold: 1
```

**Database:**
```yaml
livenessProbe:
  exec:
    command:
    - pg_isready
    - -U
    - postgres
  initialDelaySeconds: 30
  periodSeconds: 10
readinessProbe:
  exec:
    command:
    - pg_isready
    - -U
    - postgres
  initialDelaySeconds: 5
  periodSeconds: 5
```

## Cleanup

```bash
kubectl delete pod aggressive-probe slow-startup startup-probe-fixed exec-probe tcp-probe grpc-probe tuned-probes custom-health graceful-shutdown
kubectl delete deployment readiness-test probe-overhead rolling-update
kubectl delete service readiness-svc
```

---
