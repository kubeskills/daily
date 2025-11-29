# Day 9: ConfigMap Updates - The Sync Gap

## THE IDEA:
Update a ConfigMap and discover your pods don't see the changes. The kubelet
sync period, volume projection delays, and environment variable immutability
create a confusing inconsistency.

## THE SETUP:
Mount a ConfigMap as a volume and as environment variables. Update the ConfigMap
and observe which method gets updates (and how long it takes).

## WHAT I LEARNED:
- Environment variables from ConfigMaps are set at pod creation (immutable)
- Volume-mounted ConfigMaps update eventually (kubelet sync period: ~1 minute)
- subPath mounts NEVER update (broken symlink chain)
- Deployments don't auto-restart when ConfigMaps change

## WHY IT MATTERS:
ConfigMap misunderstandings cause:
- "I updated the config but nothing changed" support tickets
- Stale configuration running for hours
- Security vulnerabilities from delayed secret rotation

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-009

---

## Killercoda Lab Instructions

### Step 1: Create a ConfigMap

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: "production"
  LOG_LEVEL: "info"
  config.json: |
    {
      "feature_flag": false,
      "max_connections": 100
    }
EOF
```

### Step 2: Deploy pod with ConfigMap as env vars AND volume

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: config-test
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'while true; do echo "--- ENV VARS ---"; echo "APP_MODE=$APP_MODE"; echo "LOG_LEVEL=$LOG_LEVEL"; echo "--- VOLUME FILE ---"; cat /config/config.json; echo ""; sleep 10; done']
    env:
    - name: APP_MODE
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_MODE
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
    volumeMounts:
    - name: config-volume
      mountPath: /config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
EOF

```

**Watch the output:**

```bash
kubectl logs config-test -f

```

You'll see APP_MODE=production, LOG_LEVEL=info, and the JSON config.

Keep this running in one terminal. Open another terminal for the next step.

### Step 3: Update the ConfigMap

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: "debug"           # Changed!
  LOG_LEVEL: "verbose"        # Changed!
  config.json: |
    {
      "feature_flag": true,
      "max_connections": 500
    }
EOF

```

**Watch the logs (keep watching for 2 minutes):**

Observations:

- ENV VARS: Still show "production" and "info" (never change!)
- VOLUME FILE: Updates after ~60 seconds to show new values

### Step 4: Verify environment variables are frozen

```bash
kubectl exec config-test -- env | grep -E "APP_MODE|LOG_LEVEL"

```

Still shows old values! Environment variables are injected at pod creation.

### Step 5: Check the volume mount update

```bash
kubectl exec config-test -- cat /config/config.json

```

Shows new values (feature_flag: true, max_connections: 500).

### Step 6: Test subPath mount (broken updates)

```bash
kubectl delete pod config-test

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: subpath-test
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'while true; do echo "--- SUBPATH FILE ---"; cat /app/config.json; sleep 10; done']
    volumeMounts:
    - name: config-volume
      mountPath: /app/config.json
      subPath: config.json  # This breaks auto-updates!
  volumes:
  - name: config-volume
    configMap:
      name: app-config
EOF

```

**Watch logs:**

```bash
kubectl logs subpath-test -f

```

### Step 7: Update ConfigMap again

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: "maintenance"
  LOG_LEVEL: "error"
  config.json: |
    {
      "feature_flag": false,
      "max_connections": 10
    }
EOF

```

**Wait 2+ minutes, then check:**

```bash
kubectl exec subpath-test -- cat /app/config.json

```

STILL shows old values! subPath mounts never update.

### Step 8: Force pod restart on ConfigMap change

**Option A: Annotation hash trigger (manual)**

```bash
kubectl delete pod subpath-test

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-app
  template:
    metadata:
      labels:
        app: config-app
      annotations:
        # Change this value to force restart
        configmap-hash: "v1"
    spec:
      containers:
      - name: app
        image: busybox
        command: ['sh', '-c', 'echo "APP_MODE=$APP_MODE"; sleep 3600']
        env:
        - name: APP_MODE
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_MODE
EOF

```

**Update annotation to trigger rollout:**

```bash
kubectl patch deployment config-app -p '{"spec":{"template":{"metadata":{"annotations":{"configmap-hash":"v2"}}}}}'

```

**Watch rollout:**

```bash
kubectl rollout status deployment config-app
kubectl logs -l app=config-app

```

Now shows updated APP_MODE value!

### Step 9: Automatic hash injection with Kustomize

```bash
mkdir -p /tmp/kustomize-demo && cd /tmp/kustomize-demo

cat <<EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
configMapGenerator:
- name: app-config
  literals:
  - APP_MODE=staging
  - LOG_LEVEL=debug
EOF

cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kustomize-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kustomize-app
  template:
    metadata:
      labels:
        app: kustomize-app
    spec:
      containers:
      - name: app
        image: busybox
        command: ['sh', '-c', 'echo "APP_MODE=$APP_MODE"; sleep 3600']
        envFrom:
        - configMapRef:
            name: app-config
EOF

kubectl apply -k .

```

**Check the generated ConfigMap name:**

```bash
kubectl get configmap | grep app-config

```

Notice the hash suffix! Change the literals in kustomization.yaml and reapply -
new ConfigMap name triggers deployment rollout automatically.

Key Observations

✅ **Environment variables** - frozen at pod creation, require restart
✅ **Volume mounts** - update after ~60s (kubelet sync period)
✅ **subPath mounts** - NEVER update (known limitation)
✅ **Deployments** - don't watch ConfigMaps, need manual trigger

Production Patterns

**Sidecar for config reload:**

```yaml
containers:
- name: config-reloader
  image: jimmidyson/configmap-reload:v0.5.0
  args:
  - --volume-dir=/config
  - --webhook-url=http://localhost:8080/-/reload
  volumeMounts:
  - name: config
    mountPath: /config

```

**Reloader operator (auto-restart):**

```yaml
metadata:
  annotations:
    reloader.stakater.com/auto: "true"

```

Cleanup

```bash
kubectl delete deployment config-app kustomize-app 2>/dev/null
kubectl delete pod config-test subpath-test 2>/dev/null
kubectl delete configmap app-config 2>/dev/null
cd ~ && rm -rf /tmp/kustomize-demo

```

---

Next: Day 10 - Secret Rotation Without Restarts