## Day 76: GitOps Failures - When Deployments Don't Sync

### Email Copy

**Subject:** Day 76: GitOps Failures - When Git Push Doesn't Deploy

THE IDEA:
Push to Git and watch nothing deploy. ArgoCD out of sync, Flux reconciliation stuck,
webhook failures, and manifests that look good but won't apply.

THE SETUP:
Deploy ArgoCD/Flux, break sync loops, create invalid manifests that pass CI, test
webhook failures, and discover what breaks GitOps workflows.

WHAT I LEARNED:
- Sync failures leave cluster in unknown state
- Auto-sync can make problems worse (bad manifests everywhere)
- Pruning deletes resources not in Git (dangerous!)
- Kustomize/Helm rendering happens at sync time
- Health checks determine sync status
- Webhook auth failures block automation

WHY IT MATTERS:
GitOps failures cause:
- Manual kubectl applies (defeating automation)
- Cluster drift from Git (source of truth lost)
- Failed deployments go unnoticed
- Rollback requires Git revert + manual intervention

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-076

Tomorrow: Secret management disasters and leaked credentials.

---
Chad
Kube Daily - Week 12: DevOps & Security Deep Dive


---

## Killercoda Lab Instructions


## Step 1: Simulate GitOps repository structure

```bash
# Create local "Git repo" structure
mkdir -p /tmp/gitops-repo/{base,overlays/dev,overlays/prod}

# Base manifests
cat > /tmp/gitops-repo/base/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: nginx:1.21
        ports:
        - containerPort: 80
EOF

cat > /tmp/gitops-repo/base/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
  - port: 80
EOF

# Kustomization
cat > /tmp/gitops-repo/base/kustomization.yaml << 'EOF'
resources:
- deployment.yaml
- service.yaml
EOF

echo "GitOps repo structure created"
tree /tmp/gitops-repo 2>/dev/null || find /tmp/gitops-repo
```

## Step 2: Test manifest validation failure

```bash
# Create invalid manifest in Git
cat > /tmp/gitops-repo/base/invalid.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken
spec:
  replicas: 2
  # Missing required selector!
  template:
    metadata:
      labels:
        app: broken
    spec:
      containers:
      - name: app
        image: nginx
EOF

# Update kustomization
cat >> /tmp/gitops-repo/base/kustomization.yaml << 'EOF'
- invalid.yaml
EOF

# Try to apply
kubectl apply -k /tmp/gitops-repo/base 2>&1 || echo "Sync would fail - invalid manifest!"
```

## Step 3: Test Kustomize rendering failure

```bash
# Create overlay with wrong base path
mkdir -p /tmp/gitops-repo/overlays/staging

cat > /tmp/gitops-repo/overlays/staging/kustomization.yaml << 'EOF'
bases:
- ../../wrong-path  # Path doesn't exist!

patchesStrategicMerge:
- replica-patch.yaml
EOF

# Try to build
kubectl kustomize /tmp/gitops-repo/overlays/staging 2>&1 || echo "Kustomize build failed!"
```

## Step 4: Test resource pruning danger

```bash
# Deploy initial resources
kubectl create namespace gitops-test
kubectl apply -f /tmp/gitops-repo/base/deployment.yaml -n gitops-test
kubectl apply -f /tmp/gitops-repo/base/service.yaml -n gitops-test

# Check resources
kubectl get all -n gitops-test

# Simulate pruning: Remove service from Git
rm /tmp/gitops-repo/base/service.yaml
sed -i '/service.yaml/d' /tmp/gitops-repo/base/kustomization.yaml

echo "With auto-prune enabled:"
echo "- Service would be DELETED from cluster"
echo "- Even though it's needed by deployment"
echo "- Can cause production outages"
```

## Step 5: Test sync with wrong namespace

```bash
# Manifest specifies namespace
cat > /tmp/gitops-repo/wrong-namespace.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: config
  namespace: production  # Wrong namespace!
data:
  key: value
EOF

# Try to apply to different namespace
kubectl apply -f /tmp/gitops-repo/wrong-namespace.yaml -n gitops-test 2>&1 || echo "Namespace mismatch!"
```

## Step 6: Test circular dependency

```bash
# Create resources with circular dependency
cat > /tmp/gitops-repo/circular-a.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-a
data:
  ref: config-b
EOF

cat > /tmp/gitops-repo/circular-b.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-b
  annotations:
    depends-on: config-a  # Circular!
data:
  ref: config-a
EOF

echo "Circular dependencies cause:"
echo "- Sync loops"
echo "- Health check failures"
echo "- Resources stuck progressing"
```

## Step 7: Test image tag mismatch

```bash
# Git has one image tag
cat > /tmp/gitops-repo/image-test.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - name: app
        image: nginx:1.21  # Old version in Git
EOF

kubectl apply -f /tmp/gitops-repo/image-test.yaml -n gitops-test

# Manually update cluster to different tag
kubectl set image deployment/app-v1 app=nginx:1.22 -n gitops-test

# Cluster now drifted from Git
kubectl get deployment app-v1 -n gitops-test -o jsonpath='{.spec.template.spec.containers[0].image}'
echo ""
echo "Expected: nginx:1.21 (from Git)"
echo "Actual: nginx:1.22 (manual change)"
echo "Cluster has drifted from Git!"
```

## Step 8: Test health check failures

```bash
# Deploy app that never becomes healthy
cat > /tmp/gitops-repo/unhealthy.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: unhealthy-app
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
        image: nginx
        readinessProbe:
          httpGet:
            path: /healthz  # Endpoint doesn't exist
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
EOF

kubectl apply -f /tmp/gitops-repo/unhealthy.yaml -n gitops-test

sleep 30

# Sync shows "Progressing" forever
kubectl get deployment unhealthy-app -n gitops-test
kubectl get pods -n gitops-test -l app=unhealthy
```

## Step 9: Test secret in Git (anti-pattern)

```bash
# BAD: Secret checked into Git
cat > /tmp/gitops-repo/bad-secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: db-password
type: Opaque
data:
  password: cGFzc3dvcmQxMjM=  # base64("password123")
EOF

echo "Secrets in Git:"
echo "- Visible in Git history"
echo "- Accessible to anyone with repo access"
echo "- Can't rotate without Git commit"
echo ""
echo "Solution: Use SealedSecrets or external secret managers"
```

## Step 10: Test auto-sync with bad manifest

```bash
# Simulate auto-sync pushing broken manifest
cat > /tmp/gitops-repo/auto-sync-bad.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auto-sync-test
spec:
  replicas: 100  # Way too many!
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - name: app
        image: nginx
        resources:
          requests:
            cpu: "10"  # Unrealistic
            memory: "50Gi"
EOF

echo "With auto-sync enabled:"
echo "- Bad manifest automatically applied"
echo "- Cluster tries to schedule 100 pods"
echo "- Resource exhaustion"
echo "- No manual approval gate"

# Don't actually apply this
```

## Step 11: Test webhook authentication failure

```bash
# Simulate webhook configuration
cat > /tmp/webhook-config.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: webhook-config
data:
  webhook-url: https://gitops.example.com/webhook
  secret: wrong-secret  # Authentication fails!
EOF

echo "Webhook failures cause:"
echo "- Git push doesn't trigger sync"
echo "- Deployments delayed"
echo "- No automatic reconciliation"
echo "- Manual sync required"
```

## Step 12: Test sync wave ordering

```bash
# Resources need specific order
cat > /tmp/gitops-repo/namespace.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: app-namespace
  annotations:
    argocd.argoproj.io/sync-wave: "0"  # Create first
EOF

cat > /tmp/gitops-repo/configmap-dep.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: app-namespace  # Depends on namespace
  annotations:
    argocd.argoproj.io/sync-wave: "1"  # Create second
data:
  key: value
EOF

cat > /tmp/gitops-repo/deployment-dep.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-config
  namespace: app-namespace
  annotations:
    argocd.argoproj.io/sync-wave: "2"  # Create last
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - name: app
        image: nginx
        envFrom:
        - configMapRef:
            name: app-config
EOF

echo "Without sync waves:"
echo "- Resources applied in random order"
echo "- Deployment fails (ConfigMap doesn't exist)"
echo "- Namespace doesn't exist yet"
```

## Step 13: Test diff detection failure

```bash
# Create resource with generated fields
cat > /tmp/gitops-repo/diff-test.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: diff-test
spec:
  selector:
    app: test
  ports:
  - port: 80
    targetPort: 80
  # Kubernetes adds: clusterIP, sessionAffinity, type
EOF

kubectl apply -f /tmp/gitops-repo/diff-test.yaml -n gitops-test

# Get actual resource
kubectl get service diff-test -n gitops-test -o yaml > /tmp/actual-service.yaml

# Compare
echo "Git manifest vs Cluster:"
diff /tmp/gitops-repo/diff-test.yaml /tmp/actual-service.yaml || echo "Cluster has additional fields"

echo ""
echo "GitOps tools must ignore generated fields"
echo "Otherwise: constant out-of-sync status"
```

## Step 14: Test large manifest timeout

```bash
# Simulate very large manifest
cat > /tmp/gitops-repo/large-manifest.yaml << 'EOF'
# In production: CRD with thousands of lines
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: large.example.com
spec:
  group: example.com
  names:
    kind: Large
    plural: larges
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            # Imagine 5000 more lines of schema...
EOF

echo "Large manifests cause:"
echo "- Kubectl apply timeout"
echo "- API server timeout"
echo "- Sync marked as failed"
echo "- Need to increase timeout values"
```

## Step 15: GitOps troubleshooting guide

```bash
cat > /tmp/gitops-troubleshooting.md << 'EOF'
# GitOps Troubleshooting Guide

## Common Issues

### Sync Not Happening
- Check webhook configuration and delivery
- Test Git connectivity from controller
- Validate manifest syntax locally

### Out of Sync Status
- Compare Git to cluster with kubectl diff
- Check for generated fields causing drift
- Look for manual changes to cluster

### Sync Failures
- Check application controller logs
- Review recent events
- Validate manifest with dry-run

### Auto-prune Deleted Live Resources
- Disable auto-prune
- Use sync options: Prune=false
- Add resources back to Git if needed

### Sync Loops
- Add ignore differences for generated fields
- Fix failing health checks
- Break circular dependencies

## Best Practices
1. Validate before merge (CI pipeline)
2. Use sync waves for ordering
3. Test in staging first
4. Monitor sync status
5. Secure the pipeline
6. Have rollback plan
EOF

cat /tmp/gitops-troubleshooting.md
```

## Key Observations

✅ **Sync failures** - invalid manifests block deployment
✅ **Auto-sync** - dangerous with bad manifests
✅ **Pruning** - deletes resources not in Git
✅ **Health checks** - determine sync success
✅ **Drift** - manual changes cause out-of-sync
✅ **Webhooks** - auth failures block automation

## Production Patterns

**ArgoCD Application:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production-app
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/org/repo.git
    targetRevision: main
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: false  # Manual approval for deletions
      selfHeal: true
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas  # Ignore HPA changes
```

**Monitoring GitOps:**
```yaml
# Prometheus alerts
- alert: GitOpsSyncFailed
  expr: |
    argocd_app_sync_total{phase="Failed"} > 0
  for: 10m
  annotations:
    summary: "ArgoCD sync failed for {{ $labels.name }}"

- alert: GitOpsOutOfSync
  expr: |
    argocd_app_info{sync_status="OutOfSync"} == 1
  for: 30m
  annotations:
    summary: "Application {{ $labels.name }} out of sync > 30m"
```

## Cleanup

```bash
kubectl delete namespace gitops-test 2>/dev/null
rm -rf /tmp/gitops-repo /tmp/webhook-config.yaml /tmp/*.yaml /tmp/*.md
```

---
Next: Day 77 - Secret Management Disasters and Leaked Credentials
