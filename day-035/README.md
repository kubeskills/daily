## Day 35: Init Container Dependencies - Circular Deadlocks

## THE IDEA:
Create two deployments where each has an init container waiting for the other's
service to be ready. Watch both get stuck in Init state forever in a classic
circular dependency deadlock.

## THE SETUP:
Deploy Service A with init container checking Service B, and Service B with
init container checking Service A. Neither can start because both are waiting.

## WHAT I LEARNED:
- Init containers run sequentially before main container
- Init containers block pod startup until completion
- Circular dependencies create permanent deadlock
- Service readiness != pod readiness
- DNS resolution works even if endpoints are empty

## WHY IT MATTERS:
Init container deadlocks cause:
- Deployments stuck in Init forever
- Rollouts that never complete
- Chicken-and-egg problems in microservices

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-035

Tomorrow: Job completion issues and backoff limits.


---

# Killercoda Lab Instructions


## Step 1: Create Service A (depends on Service B)
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-a
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
      initContainers:
      - name: wait-for-service-b
        image: busybox
        command: ['sh', '-c']
        args:
        - |
          echo "Waiting for service-b..."
          until nslookup service-b; do
            echo "service-b not ready yet"
            sleep 2
          done
          echo "service-b DNS resolved, checking endpoint..."
          until wget -O- http://service-b:80; do
            echo "service-b not responding yet"
            sleep 2
          done
          echo "service-b is ready!"
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-text=Service A"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: service-a
spec:
  selector:
    app: service-a
  ports:
  - port: 80
    targetPort: 5678
EOF
```

## Step 2: Create Service B (depends on Service A)
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-b
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
      initContainers:
      - name: wait-for-service-a
        image: busybox
        command: ['sh', '-c']
        args:
        - |
          echo "Waiting for service-a..."
          until nslookup service-a; do
            echo "service-a not ready yet"
            sleep 2
          done
          echo "service-a DNS resolved, checking endpoint..."
          until wget -O- http://service-a:80; do
            echo "service-a not responding yet"
            sleep 2
          done
          echo "service-a is ready!"
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-text=Service B"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: service-b
spec:
  selector:
    app: service-b
  ports:
  - port: 80
    targetPort: 5678
EOF
```

## Step 3: Watch the deadlock
```bash
kubectl get pods -w
```

Both stuck in Init:0/1 forever!

## Step 4: Check init container logs
```bash
# Service A waiting for Service B
kubectl logs -l app=service-a -c wait-for-service-b

# Service B waiting for Service A
kubectl logs -l app=service-b -c wait-for-service-a
```

Both endlessly waiting!

## Step 5: Describe pods to see the issue
```bash
kubectl describe pod -l app=service-a | grep -A 10 "Init Containers:"
kubectl describe pod -l app=service-b | grep -A 10 "Init Containers:"
```

## Step 6: Fix - remove init container from one service
```bash
kubectl delete deployment service-a

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-a
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
      # No init container - break the cycle!
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-text=Service A"]
        ports:
        - containerPort: 5678
EOF
```

**Watch them start:**
```bash
kubectl get pods -w
```

Service A starts → Service B's init completes → Service B starts!

## Step 7: Test init container with timeout
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: init-timeout-test
spec:
  initContainers:
  - name: wait-with-timeout
    image: busybox
    command: ['sh', '-c']
    args:
    - |
      echo "Waiting up to 30 seconds..."
      timeout 30 sh -c 'until nslookup nonexistent-service; do sleep 2; done' || {
        echo "Timeout reached, continuing anyway"
        exit 0
      }
  containers:
  - name: app
    image: nginx
EOF
```

**Check logs:**
```bash
kubectl logs init-timeout-test -c wait-with-timeout
```

Init completes after timeout!

## Step 8: Test multiple init containers (sequential)
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multi-init
spec:
  initContainers:
  - name: init-1
    image: busybox
    command: ['sh', '-c', 'echo "Init 1 running"; sleep 5; echo "Init 1 done"']
  - name: init-2
    image: busybox
    command: ['sh', '-c', 'echo "Init 2 running"; sleep 5; echo "Init 2 done"']
  - name: init-3
    image: busybox
    command: ['sh', '-c', 'echo "Init 3 running"; sleep 5; echo "Init 3 done"']
  containers:
  - name: app
    image: nginx
EOF
```

**Watch sequential execution:**
```bash
kubectl get pod multi-init -w
```

Init:0/3 → Init:1/3 → Init:2/3 → Init:3/3 → Running

## Step 9: Test init container failure blocking main
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: init-fail
spec:
  initContainers:
  - name: failing-init
    image: busybox
    command: ['sh', '-c', 'echo "Failing intentionally"; exit 1']
  containers:
  - name: app
    image: nginx
EOF
```

**Check status:**
```bash
kubectl get pod init-fail
```

Stuck in Init:CrashLoopBackOff - main container never starts!

## Step 10: Test init container with retries
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: init-retry
spec:
  restartPolicy: OnFailure
  initContainers:
  - name: retry-init
    image: busybox
    command: ['sh', '-c']
    args:
    - |
      if [ -f /tmp/success ]; then
        echo "Success file exists, continuing"
        exit 0
      else
        echo "First attempt, failing"
        touch /tmp/success
        exit 1
      fi
    volumeMounts:
    - name: state
      mountPath: /tmp
  containers:
  - name: app
    image: nginx
  volumes:
  - name: state
    emptyDir: {}
EOF
```

**Watch retry behavior:**
```bash
kubectl get pod init-retry -w
kubectl describe pod init-retry | grep -A 10 Events
```

## Step 11: Test safe dependency pattern (check without blocking)
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: startup-script
data:
  startup.sh: |
    #!/bin/sh
    echo "Main app starting..."

    # Non-blocking check in background
    (
      echo "Checking for optional dependency..."
      for i in $(seq 1 10); do
        if wget -O- --timeout=2 http://service-a:80 2>/dev/null; then
          echo "Dependency available!"
          break
        fi
        echo "Dependency not ready, will retry..."
        sleep 5
      done
    ) &

    # Start main app immediately
    nginx -g "daemon off;"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: safe-dependency
spec:
  replicas: 1
  selector:
    matchLabels:
      app: safe-dep
  template:
    metadata:
      labels:
        app: safe-dep
    spec:
      containers:
      - name: app
        image: nginx
        command: ["/bin/sh", "/scripts/startup.sh"]
        volumeMounts:
        - name: scripts
          mountPath: /scripts
      volumes:
      - name: scripts
        configMap:
          name: startup-script
          defaultMode: 0755
EOF
```

App starts immediately, checks dependency asynchronously!

## Step 12: Test readiness probe vs init container
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: readiness-vs-init
spec:
  initContainers:
  - name: quick-init
    image: busybox
    command: ['sh', '-c', 'echo "Init complete"; sleep 2']
  containers:
  - name: app
    image: hashicorp/http-echo
    args: ["-text=Ready"]
    ports:
    - containerPort: 5678
    readinessProbe:
      httpGet:
        path: /
        port: 5678
      initialDelaySeconds: 5
      periodSeconds: 2
EOF
```

**Watch progression:**
```bash
kubectl get pod readiness-vs-init -w
```

Init:0/1 → Running (not Ready) → Running (Ready)

## Step 13: Test Job with init container
```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: job-with-init
spec:
  template:
    spec:
      initContainers:
      - name: setup
        image: busybox
        command: ['sh', '-c', 'echo "Setting up..."; sleep 3']
      containers:
      - name: worker
        image: busybox
        command: ['sh', '-c', 'echo "Job running"; sleep 5; echo "Job done"']
      restartPolicy: Never
EOF
```

**Check completion:**
```bash
kubectl get job job-with-init -w
kubectl logs job/job-with-init -c setup
kubectl logs job/job-with-init -c worker
```

## Step 14: Test StatefulSet init container ordering
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: stateful-init
spec:
  serviceName: stateful-init
  replicas: 3
  selector:
    matchLabels:
      app: stateful-init
  template:
    metadata:
      labels:
        app: stateful-init
    spec:
      initContainers:
      - name: check-predecessor
        image: busybox
        command: ['sh', '-c']
        args:
        - |
          ordinal=${HOSTNAME##*-}
          if [ $ordinal -eq 0 ]; then
            echo "I am pod-0, no predecessor to check"
            exit 0
          fi
          predecessor=$((ordinal - 1))
          echo "Checking for stateful-init-$predecessor..."
          until nslookup stateful-init-$predecessor.stateful-init; do
            echo "Predecessor not ready"
            sleep 2
          done
          echo "Predecessor ready!"
      containers:
      - name: app
        image: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: stateful-init
spec:
  clusterIP: None
  selector:
    app: stateful-init
  ports:
  - port: 80
EOF
```

**Watch ordered startup:**
```bash
kubectl get pods -l app=stateful-init -w
```

pod-0 starts → pod-1 waits → pod-1 starts → pod-2 waits → pod-2 starts

## Step 15: Debug stuck init containers
```bash
# Find pods stuck in Init
kubectl get pods --all-namespaces --field-selector=status.phase=Pending | grep Init

# Check which init container is stuck
kubectl get pod init-fail -o jsonpath='{.status.initContainerStatuses[*].name}'
echo ""

# Check init container state
kubectl get pod init-fail -o jsonpath='{.status.initContainerStatuses[*].state}'
echo ""

# Get logs from specific init container
kubectl logs init-fail -c failing-init
```

## Key Observations

✅ **Sequential execution** - init containers run in order
✅ **Blocking behavior** - main container waits for all inits
✅ **Circular dependencies** - create permanent deadlock
✅ **Restart policy** - applies to init containers too
✅ **Timeout patterns** - prevent infinite waiting
✅ **StatefulSet ordering** - each pod can wait for predecessor

## Production Patterns

**Safe dependency check with timeout:**
```yaml
initContainers:
- name: wait-for-db
  image: busybox
  command: ['sh', '-c']
  args:
  - |
    timeout 300 sh -c 'until nc -z postgres 5432; do sleep 2; done' || {
      echo "Timeout waiting for postgres, starting anyway"
      exit 0
    }
```

**Database migration init:**
```yaml
initContainers:
- name: run-migrations
  image: myapp:migrations
  command: ['./migrate', 'up']
  env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: url
```

**Configuration download:**
```yaml
initContainers:
- name: fetch-config
  image: curlimages/curl
  command: ['sh', '-c']
  args:
  - curl -o /config/app.yaml https://config-server/app/prod
  volumeMounts:
  - name: config
    mountPath: /config
containers:
- name: app
  volumeMounts:
  - name: config
    mountPath: /app/config
volumes:
- name: config
  emptyDir: {}
```

**Ordered startup in StatefulSet:**
```yaml
initContainers:
- name: wait-for-master
  image: busybox
  command: ['sh', '-c']
  args:
  - |
    if [ "${HOSTNAME##*-}" = "0" ]; then
      echo "I am the master"
      exit 0
    fi
    until nslookup ${STATEFUL_SET_NAME}-0.${SERVICE_NAME}; do
      sleep 2
    done
```

## Cleanup
```bash
kubectl delete deployment service-a service-b safe-dependency
kubectl delete service service-a service-b
kubectl delete pod init-timeout-test multi-init init-fail init-retry readiness-vs-init
kubectl delete job job-with-init
kubectl delete statefulset stateful-init
kubectl delete service stateful-init
kubectl delete configmap startup-script
```

---
