## Day 32: ArgoCD Sync - Out of Sync Nightmares


## THE IDEA:
Deploy ArgoCD, point it at a Git repo with Kubernetes manifests, and introduce 
sync failures from validation errors, pruning conflicts, and sync waves gone 
wrong.

## THE SETUP:
Install ArgoCD, create an Application pointing to a repo, trigger sync, and 
watch it fail from resource hooks, health checks, and manual changes.

## WHAT I LEARNED:
- ArgoCD compares Git (desired) vs cluster (actual) state
- Sync can fail on pre-sync hooks
- Manual kubectl changes cause "OutOfSync" status
- Prune deletes resources not in Git
- Health checks determine sync success

## WHY IT MATTERS:
GitOps sync issues cause:
- Deployments stuck in "Progressing" forever
- Manual hotfixes overwritten by Git
- Resource deletion when prune is enabled

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-032

Tomorrow: Service mesh traffic routing failures.

---

# Killercoda Lab Instructions


## Step 1: Install ArgoCD

```bash
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
```

## Step 2: Access ArgoCD UI (port-forward)

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 > /dev/null 2>&1 &

# Get admin password
ARGOCD_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
echo "ArgoCD Password: $ARGOCD_PASSWORD"
```

**Login with CLI:**
```bash
# Install ArgoCD CLI
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd

argocd login localhost:8080 --username admin --password $ARGOCD_PASSWORD --insecure
```

## Step 3: Create Git repo (simulated with local path)

```bash
mkdir -p /tmp/gitops-repo
cd /tmp/gitops-repo

cat > deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
EOF

cat > service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: demo-app
  namespace: default
spec:
  selector:
    app: demo
  ports:
  - port: 80
    targetPort: 80
EOF
```

## Step 4: Create ArgoCD Application

```bash
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: /tmp/gitops-repo
    targetRevision: HEAD
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: false
      selfHeal: false
EOF
```

**Note:** In production, use real Git repo URL

## Step 5: Check Application status

```bash
kubectl get application -n argocd demo-app

argocd app get demo-app
```

Status: OutOfSync (Git different from cluster)

## Step 6: Manual sync

```bash
argocd app sync demo-app
```

**Verify resources created:**
```bash
kubectl get deployment demo-app
kubectl get service demo-app
```

## Step 7: Cause OutOfSync - manual kubectl change

```bash
# Manually scale deployment
kubectl scale deployment demo-app --replicas=5

# Check ArgoCD status
argocd app get demo-app
```

Status: OutOfSync (cluster drift from Git)

## Step 8: Enable auto-sync and self-heal

```bash
kubectl patch application demo-app -n argocd --type=merge -p '{
  "spec": {
    "syncPolicy": {
      "automated": {
        "prune": false,
        "selfHeal": true
      }
    }
  }
}'

# Watch it revert to 2 replicas
kubectl get deployment demo-app -w
```

Self-heal reverted manual change!

## Step 9: Test sync failure with invalid YAML

```bash
cd /tmp/gitops-repo

cat > broken.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: broken
  namespace: default
data:
  key: value
  # Invalid: missing colon
  invalid
EOF

# Trigger sync
argocd app sync demo-app 2>&1 || echo "Sync failed"
```

**Error:**
```
error unmarshaling JSON: while decoding JSON: json: cannot unmarshal...
```

## Step 10: Fix and retry

```bash
rm broken.yaml

# Sync succeeds
argocd app sync demo-app
```

## Step 11: Test with resource hooks

```bash
cat > pre-sync-job.yaml << 'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: pre-sync-job
  namespace: default
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: pre-sync
        image: busybox
        command: ['sh', '-c', 'echo "Running pre-sync"; sleep 5; exit 0']
      restartPolicy: Never
  backoffLimit: 0
EOF

argocd app sync demo-app
```

Pre-sync job runs before deployment!

## Step 12: Test failed pre-sync hook

```bash
cat > pre-sync-job.yaml << 'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: pre-sync-fail-job
  namespace: default
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: fail
        image: busybox
        command: ['sh', '-c', 'echo "Failing intentionally"; exit 1']
      restartPolicy: Never
  backoffLimit: 0
EOF

argocd app sync demo-app 2>&1 || echo "Sync blocked by hook failure"
```

Sync fails! Pre-sync hook must succeed.

## Step 13: Remove failed hook, test sync waves

```bash
rm pre-sync-job.yaml

cat > wave-1.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: wave-1
  namespace: default
  annotations:
    argocd.argoproj.io/sync-wave: "1"
data:
  order: "first"
EOF

cat > wave-2.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: wave-2
  namespace: default
  annotations:
    argocd.argoproj.io/sync-wave: "2"
data:
  order: "second"
EOF

argocd app sync demo-app --dry-run
```

Shows sync order!

## Step 14: Test prune behavior

```bash
# Enable prune
kubectl patch application demo-app -n argocd --type=merge -p '{
  "spec": {
    "syncPolicy": {
      "automated": {
        "prune": true,
        "selfHeal": true
      }
    }
  }
}'

# Create resource manually
kubectl create configmap manual-cm --from-literal=key=value

# Wait for sync
sleep 10

# Check if pruned
kubectl get configmap manual-cm 2>&1 || echo "Pruned by ArgoCD"
```

Manual ConfigMap deleted (not in Git)!

## Step 15: Test health check failure

```bash
cat > unhealthy.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: unhealthy-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: unhealthy
  template:
    metadata:
      labels:
        app: unhealthy
    spec:
      containers:
      - name: app
        image: nonexistent-image:latest
        imagePullPolicy: Always
EOF

argocd app sync demo-app

# Check status
argocd app get demo-app
```

App shows "Progressing" - health check waiting for pod

## Step 16: Test sync with replace strategy

```bash
# Remove unhealthy app
rm unhealthy.yaml

cat > deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: default
  annotations:
    argocd.argoproj.io/sync-options: Replace=true
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.26
EOF

argocd app sync demo-app
```

Replace strategy: delete then create (vs default patch)

## Step 17: Test selective sync

```bash
# Sync only specific resource
argocd app sync demo-app --resource=Service:default/demo-app
```

## Step 18: Check sync history

```bash
argocd app history demo-app
```

Shows all sync revisions.

## Step 19: Rollback via ArgoCD

```bash
# Get history
REVISION=$(argocd app history demo-app | grep -v REVISION | head -1 | awk '{print $1}')

# Rollback
argocd app rollback demo-app $REVISION
```

## Step 20: Test app deletion with cascade

```bash
# Delete app (keeps resources)
argocd app delete demo-app --cascade=false

# Resources still there
kubectl get deployment demo-app

# Delete with cascade
kubectl delete application demo-app -n argocd

# Deployment deleted too
kubectl get deployment demo-app 2>&1 || echo "Deployment deleted"
```

## Key Observations

✅ **OutOfSync** - Git != Cluster state
✅ **Auto-sync** - automatically syncs on Git changes
✅ **Self-heal** - reverts manual kubectl changes
✅ **Prune** - deletes resources not in Git
✅ **Hooks** - PreSync/Sync/PostSync/SyncFail
✅ **Sync waves** - control resource creation order
✅ **Health checks** - determine sync success

## Production Patterns

**Application with sync policies:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/repo
    targetRevision: main
    path: apps/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

**Sync waves for dependencies:**
```yaml
# Database first (wave 0)
apiVersion: v1
kind: ConfigMap
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"

# Migration job (wave 1)
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"
    argocd.argoproj.io/hook: Sync

# Application (wave 2)
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "2"
```

**Resource hooks:**
```yaml
# Pre-deployment backup
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation

# Post-deployment smoke test
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

## Cleanup

```bash
pkill -f "port-forward.*argocd"
kubectl delete namespace argocd
rm -rf /tmp/gitops-repo
```

---
Next: Day 33 - Service Mesh Traffic Routing Failures