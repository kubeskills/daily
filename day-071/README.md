## Day 71: Operator Failures - Custom Controllers Breaking

### Email Copy

**Subject:** Day 71: Operator Failures - When Custom Controllers Fail

THE IDEA:
Deploy Kubernetes operators and watch them fail to reconcile resources. CRD validation
errors, operator crashes, stuck finalizers, and orphaned resources everywhere.

THE SETUP:
Install operators with wrong RBAC, corrupt CRDs, create resources the operator can't
handle, and test what breaks when custom controllers fail.

WHAT I LEARNED:
- Operators extend Kubernetes with custom resources (CRDs)
- Finalizers prevent deletion until cleanup complete
- Stuck finalizers block namespace deletion
- Operator needs RBAC for all resources it manages
- CRD validation happens at API level
- Operator crashes leave resources in unknown state

WHY IT MATTERS:
Operator failures cause:
- Resources stuck in deletion (finalizer never runs)
- Namespaces can't be deleted (stuck Terminating)
- New CRs rejected by validation
- Cluster state drift (operator not reconciling)

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-071

Tomorrow: Cluster upgrade failures and version skew.

---
Chad


---

## Killercoda Lab Instructions


## Step 1: Create simple CRD

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: webapps.example.com
spec:
  group: example.com
  names:
    kind: WebApp
    plural: webapps
    singular: webapp
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
            required: ["replicas", "image"]
            properties:
              replicas:
                type: integer
                minimum: 1
                maximum: 10
              image:
                type: string
          status:
            type: object
            properties:
              ready:
                type: boolean
    subresources:
      status: {}
EOF

# Check CRD
kubectl get crd webapps.example.com
```

## Step 2: Create custom resource

```bash
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: WebApp
metadata:
  name: my-webapp
spec:
  replicas: 3
  image: nginx:latest
EOF

# Check CR
kubectl get webapps
kubectl describe webapp my-webapp
```

## Step 3: Test CRD validation failure

```bash
# Try to create CR that violates validation
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: WebApp
metadata:
  name: invalid-webapp
spec:
  replicas: 20  # Exceeds maximum of 10!
  image: nginx
EOF

# Validation blocks creation
sleep 5
kubectl get webapp invalid-webapp 2>&1 || echo "Validation failed"
```

## Step 4: Test missing required field

```bash
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: WebApp
metadata:
  name: missing-field
spec:
  replicas: 3
  # Missing required 'image' field!
EOF

# Rejected by validation
```

## Step 5: Deploy broken operator

```bash
# Create namespace for operator
kubectl create namespace webapp-system

# Create ServiceAccount with insufficient RBAC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: webapp-operator
  namespace: webapp-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: webapp-operator
rules:
- apiGroups: ["example.com"]
  resources: ["webapps"]
  verbs: ["get", "list", "watch"]
  # Missing: create, update, patch, delete
- apiGroups: ["example.com"]
  resources: ["webapps/status"]
  verbs: ["get"]
  # Missing: update, patch
# Missing: RBAC for deployments, services
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: webapp-operator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: webapp-operator
subjects:
- kind: ServiceAccount
  name: webapp-operator
  namespace: webapp-system
EOF

# Deploy operator with insufficient permissions
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-operator
  namespace: webapp-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp-operator
  template:
    metadata:
      labels:
        app: webapp-operator
    spec:
      serviceAccountName: webapp-operator
      containers:
      - name: operator
        image: bash:5
        command:
        - bash
        - -c
        - |
          echo "Starting operator..."
          TOKEN=\$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
          API=https://kubernetes.default.svc

          while true; do
            # Watch for WebApp resources
            WEBAPPS=\$(curl -s -k -H "Authorization: Bearer \$TOKEN" \
              \$API/apis/example.com/v1/webapps)

            echo "Found webapps:"
            echo "\$WEBAPPS" | grep -o '"name":"[^"]*"' || echo "None"

            # Try to create deployment (will fail - no RBAC)
            echo "Attempting to create deployment..."
            curl -s -k -X POST \
              -H "Authorization: Bearer \$TOKEN" \
              -H "Content-Type: application/json" \
              \$API/apis/apps/v1/namespaces/default/deployments \
              -d '{"metadata":{"name":"test"},"spec":{"replicas":1,"selector":{"matchLabels":{"app":"test"}},"template":{"metadata":{"labels":{"app":"test"}},"spec":{"containers":[{"name":"app","image":"nginx"}]}}}}' \
              2>&1 | grep -o "forbidden\|created" || echo "Failed"

            sleep 10
          done
EOF

kubectl wait --for=condition=Ready pod -n webapp-system -l app=webapp-operator --timeout=60s
```

## Step 6: Check operator logs for RBAC errors

```bash
sleep 15
kubectl logs -n webapp-system -l app=webapp-operator --tail=20 | grep -i "forbidden\|error"
```

Shows permission denied errors!

## Step 7: Add finalizer to CR

```bash
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: WebApp
metadata:
  name: with-finalizer
  finalizers:
  - webapp.example.com/cleanup
spec:
  replicas: 2
  image: nginx
EOF

# Try to delete (will get stuck)
kubectl delete webapp with-finalizer &
DELETE_PID=$!

sleep 10

# Check status (stuck in Terminating)
kubectl get webapp with-finalizer
kubectl describe webapp with-finalizer | grep -A 5 "Finalizers:"

# Kill background delete
kill $DELETE_PID 2>/dev/null
```

## Step 8: Force remove finalizer

```bash
# Remove finalizer to allow deletion
kubectl patch webapp with-finalizer -p '{"metadata":{"finalizers":[]}}' --type=merge

# Now deletion completes
kubectl get webapp with-finalizer 2>&1 || echo "Deleted"
```

## Step 9: Test operator crash

```bash
# Create operator that crashes
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crashy-operator
  namespace: webapp-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: crashy-operator
  template:
    metadata:
      labels:
        app: crashy-operator
    spec:
      containers:
      - name: operator
        image: bash:5
        command:
        - bash
        - -c
        - |
          echo "Starting operator..."
          sleep 5
          echo "Crashing!"
          exit 1
EOF

# Watch crash loop
kubectl get pods -n webapp-system -l app=crashy-operator -w
```

## Step 10: Test namespace stuck in Terminating

```bash
# Create namespace with finalizer
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: stuck-namespace
  finalizers:
  - kubernetes.io/some-finalizer
EOF

# Create resources
kubectl create deployment test -n stuck-namespace --image=nginx

# Try to delete namespace
kubectl delete namespace stuck-namespace &
NS_DELETE_PID=$!

sleep 10

# Check status (stuck)
kubectl get namespace stuck-namespace

# Kill background process
kill $NS_DELETE_PID 2>/dev/null

# Force delete namespace
kubectl get namespace stuck-namespace -o json | \
  jq '.spec.finalizers = []' | \
  kubectl replace --raw /api/v1/namespaces/stuck-namespace/finalize -f -

# Now it deletes
kubectl get namespace stuck-namespace 2>&1 || echo "Namespace deleted"
```

## Step 11: Test CRD with conversion webhook

```bash
# CRD with multiple versions
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: multiversion.example.com
spec:
  group: example.com
  names:
    kind: MultiVersion
    plural: multiversions
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: false
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              oldField:
                type: string
  - name: v2
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              newField:
                type: string
  conversion:
    strategy: Webhook
    webhook:
      clientConfig:
        service:
          name: conversion-webhook
          namespace: default
          path: /convert
      conversionReviewVersions: ["v1"]
EOF

# Webhook doesn't exist - conversions fail
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: MultiVersion
metadata:
  name: test-v1
spec:
  oldField: "value"
EOF

sleep 5
kubectl get multiversion test-v1 2>&1 || echo "Conversion failed"
```

## Step 12: Test operator watch errors

```bash
# Simulate operator watching wrong resource
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wrong-watch-operator
  namespace: webapp-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wrong-watch
  template:
    metadata:
      labels:
        app: wrong-watch
    spec:
      serviceAccountName: webapp-operator
      containers:
      - name: operator
        image: bash:5
        command:
        - bash
        - -c
        - |
          TOKEN=\$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
          API=https://kubernetes.default.svc

          # Watch non-existent resource
          echo "Watching fake resource..."
          while true; do
            curl -s -k -H "Authorization: Bearer \$TOKEN" \
              \$API/apis/fake.io/v1/fakeresources?watch=true 2>&1 | \
              grep -o "404\|Not Found" || true
            sleep 5
          done
EOF

kubectl wait --for=condition=Ready pod -n webapp-system -l app=wrong-watch --timeout=60s
sleep 10
kubectl logs -n webapp-system -l app=wrong-watch --tail=10
```

## Step 13: Test operator reconciliation failure

```bash
# Create CR that operator can't handle
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: WebApp
metadata:
  name: unreconcilable
spec:
  replicas: 3
  image: "invalid@image@format"
EOF

# Operator would fail to reconcile this
kubectl get webapp unreconcilable
kubectl describe webapp unreconcilable
```

## Step 14: Test CRD deletion with existing CRs

```bash
# Create CR
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: WebApp
metadata:
  name: test-deletion
spec:
  replicas: 1
  image: nginx
EOF

# Try to delete CRD (blocked while CRs exist)
kubectl delete crd webapps.example.com &
CRD_DELETE_PID=$!

sleep 5

# CRD stuck in Terminating
kubectl get crd webapps.example.com

# Kill background process
kill $CRD_DELETE_PID 2>/dev/null

# Delete CRs first
kubectl delete webapp --all

# Now CRD can be deleted
kubectl delete crd webapps.example.com
```

## Step 15: Diagnose operator issues

```bash
cat > /tmp/operator-diagnosis.sh << 'EOF'
#!/bin/bash
echo "=== Operator Diagnosis ==="

echo -e "\n1. Custom Resource Definitions:"
kubectl get crd

echo -e "\n2. CRs with Finalizers:"
kubectl get crd -o json | jq -r '.items[].metadata.name' | while read crd; do
  plural=$(kubectl get crd $crd -o jsonpath='{.spec.names.plural}')
  kubectl get $plural -A -o json 2>/dev/null | jq -r --arg crd "$crd" '
    .items[] |
    select(.metadata.finalizers != null and (.metadata.finalizers | length) > 0) |
    "\(.metadata.namespace)/\(.metadata.name) [\($crd)]: \(.metadata.finalizers | join(", "))"
  '
done | head -10

echo -e "\n3. Resources Stuck in Terminating:"
kubectl get all -A | grep Terminating

echo -e "\n4. Namespaces Stuck in Terminating:"
kubectl get namespaces | grep Terminating

echo -e "\n5. Operator Pods Status:"
kubectl get pods -A -l 'app in (operator, controller, manager)' -o wide 2>/dev/null | head -10

echo -e "\n6. Recent Operator Events:"
kubectl get events -A --sort-by='.lastTimestamp' | grep -i "operator\|controller\|crd" | tail -10

echo -e "\n7. Common Operator Issues:"
echo "   - Missing RBAC: Operator can't manage resources"
echo "   - CRD validation: Invalid CRs rejected"
echo "   - Stuck finalizers: Resources can't be deleted"
echo "   - Operator crash: Resources not reconciled"
echo "   - Watch errors: Operator can't see changes"
echo "   - Conversion webhook down: Multi-version CRDs fail"
EOF

chmod +x /tmp/operator-diagnosis.sh
/tmp/operator-diagnosis.sh
```

## Key Observations

✅ **CRD validation** - enforced at API server level
✅ **Finalizers** - block deletion until removed
✅ **RBAC required** - operator needs permissions for all managed resources
✅ **Status subresource** - separate from spec updates
✅ **Operator crashes** - leave resources in unknown state
✅ **CRD deletion** - blocked by existing CRs

## Production Patterns

**Complete operator RBAC:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-operator
  namespace: operator-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: myapp-operator
rules:
# Custom resources
- apiGroups: ["myapp.example.com"]
  resources: ["myapps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["myapp.example.com"]
  resources: ["myapps/status"]
  verbs: ["get", "update", "patch"]
- apiGroups: ["myapp.example.com"]
  resources: ["myapps/finalizers"]
  verbs: ["update"]
# Managed resources
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# Events
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "patch"]
# Leader election
- apiGroups: ["coordination.k8s.io"]
  resources: ["leases"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: myapp-operator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: myapp-operator
subjects:
- kind: ServiceAccount
  name: myapp-operator
  namespace: operator-system
```

**CRD with proper validation:**
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: myapps.example.com
spec:
  group: example.com
  names:
    kind: MyApp
    plural: myapps
    singular: myapp
    shortNames: ["ma"]
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        required: ["spec"]
        properties:
          spec:
            type: object
            required: ["replicas", "image"]
            properties:
              replicas:
                type: integer
                minimum: 1
                maximum: 100
              image:
                type: string
                pattern: '^[a-z0-9-./:]+(:[a-z0-9-.]+)?$'
          status:
            type: object
            properties:
              phase:
                type: string
                enum: ["Pending", "Running", "Failed", "Succeeded"]
              ready:
                type: boolean
    subresources:
      status: {}
    additionalPrinterColumns:
    - name: Replicas
      type: integer
      jsonPath: .spec.replicas
    - name: Ready
      type: boolean
      jsonPath: .status.ready
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
```

**Monitoring operators:**
```yaml
# Prometheus alerts
- alert: OperatorDown
  expr: up{job="operator"} == 0
  for: 5m
  annotations:
    summary: "Operator {{ $labels.job }} is down"

- alert: CRStuckDeleting
  expr: |
    time() - kube_customresource_deletion_timestamp > 600
  annotations:
    summary: "Custom resource stuck deleting for > 10 min"

- alert: OperatorHighErrorRate
  expr: |
    rate(controller_runtime_reconcile_errors_total[5m]) > 0.1
  annotations:
    summary: "Operator error rate high"
```

## Cleanup

```bash
kubectl delete namespace webapp-system stuck-namespace 2>/dev/null
kubectl delete crd webapps.example.com multiversion.example.com 2>/dev/null
kubectl delete clusterrole webapp-operator
kubectl delete clusterrolebinding webapp-operator
rm -f /tmp/operator-diagnosis.sh
```

---
Next: Day 72 - Cluster Upgrade Failures and Version Skew
