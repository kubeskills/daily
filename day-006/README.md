# Day 6: Liveness Probes - The Restart Loop of Death

## THE IDEA:
Misconfigure a liveness probe with a 1-second timeout on a slow endpoint. Watch
the pod enter an endless restart loop even though the app is healthy.

## THE SETUP:
Deploy a web server that takes 5 seconds to respond to health checks. Configure
a liveness probe with 1-second timeout and observe the restart cascade.

## WHAT I LEARNED:
- Liveness probe failures trigger restarts (not just marking unhealthy)
- successThreshold for liveness is always 1 (by design)
- failureThreshold determines how many failures before restart
- Restart backoff can delay recovery: 10s, 20s, 40s… up to 5 minutes

## WHY IT MATTERS:
Bad liveness probes are the #1 cause of self-inflicted outages:

- Slow database queries during startup kill pods repeatedly
- High CPU load makes probes timeout, triggering more load
- Too-aggressive probes prevent apps from ever becoming healthy

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-006


---


## Killercoda Lab Instructions

### STEP 1: Deploy a slow-starting pod with a bad liveness probe:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: slow-app
spec:
  containers:
  - name: app
    image: busybox:1.36.1
    command: ["/bin/sh", "-c", "sleep 10 && mkdir -p /www && echo 'Slow response' > /www/index.html && httpd -f -p 8080 -h /www"]
    ports:
    - containerPort: 8080
    livenessProbe:
      httpGet:
        path: /
        port: 8080
      initialDelaySeconds: 2
      periodSeconds: 3
      timeoutSeconds: 1   # Too aggressive!
      failureThreshold: 2
EOF

```

**Watch the restart loop:**

```bash
kubectl get pods slow-app -w

```

You’ll see: Running → Running (health check failing) → Restart → CrashLoopBackOff

### **Step 2: Check why it’s failing**

```bash
kubectl describe pod slow-app | grep -A 10 Events

```

Look for: “Liveness probe failed: Get… context deadline exceeded”

### **Step 3: Check restart count**

```bash
kubectl get pod slow-app

```

RESTARTS column keeps incrementing!

### **Step 4: Fix the liveness probe**

```bash
kubectl delete pod slow-app

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: slow-app-fixed
spec:
  containers:
  - name: app
    image: busybox:1.36.1
    command: ["/bin/sh", "-c", "sleep 10 && mkdir -p /www && echo 'Healthy now!' > /www/index.html && httpd -f -p 8080 -h /www"]
    ports:
    - containerPort: 8080
    livenessProbe:
      httpGet:
        path: /
        port: 8080
      initialDelaySeconds: 15    # Wait for startup
      periodSeconds: 10          # Check every 10s
      timeoutSeconds: 5          # Generous timeout
      failureThreshold: 3        # Allow 3 failures
EOF

```

**Verify it stays healthy:**

```bash
kubectl get pods slow-app-fixed -w

```

Should reach Running and stay there!

### **Step 5: Compare with readiness probe**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: readiness-test
  labels:
    app: web
spec:
  containers:
  - name: app
    image: hashicorp/http-echo
    args:
    - "-text=Ready"
    - "-listen=:8080"
    readinessProbe:
      httpGet:
        path: /
        port: 8080
      initialDelaySeconds: 2
      periodSeconds: 3
      timeoutSeconds: 1
      failureThreshold: 2
    livenessProbe:
      httpGet:
        path: /
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
EOF

```

**Check endpoints:**

```bash
kubectl get endpoints web-svc -w

```

Readiness probe failure removes pod from service endpoints but doesn’t restart it!

### **Step 6: Simulate startup probe for slow-starting apps**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: slow-startup
spec:
  containers:
  - name: app
    image: busybox:1.36.1
    command: ["/bin/sh", "-c", "sleep 60 && mkdir -p /www && echo 'Finally started' > /www/index.html && httpd -f -p 8080 -h /www"]  # Simulate slow startup
    startupProbe:
      httpGet:
        path: /
        port: 8080
      periodSeconds: 5
      failureThreshold: 30  # 30 * 5s = 150s max startup time
    livenessProbe:
      httpGet:
        path: /
        port: 8080
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
EOF

```

Startup probe gives the app up to 150 seconds to start. After it succeeds once,
liveness probe takes over with stricter checks.

**Key Observations**

✅ **Liveness** - restarts container on failure
✅ **Readiness** - removes from service endpoints on failure (no restart)
✅ **Startup** - gives slow apps more time before liveness kicks in
✅ **initialDelaySeconds** - critical for apps with long initialization

**Production Guidelines**

**Conservative liveness probe (database):**

```yaml
livenessProbe:
  tcpSocket:
    port: 5432
  initialDelaySeconds: 60
  periodSeconds: 20
  timeoutSeconds: 10
  failureThreshold: 3

```

**Aggressive readiness (web server):**

```yaml
readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 2
  timeoutSeconds: 1
  failureThreshold: 1

```

**Startup probe for JVM apps:**

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 10
  failureThreshold: 30  # 5 minutes max startup

```

### Cleanup

```bash
kubectl delete pod slow-app slow-app-fixed readiness-test slow-startup
kubectl delete service web-svc

```

---

[Next: Day 7 - Init Container Failures](../day-007/README.md)
