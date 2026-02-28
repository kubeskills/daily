## Day 55: RBAC Misconfigurations - Permission Denied Hell

### Email Copy

**Subject:** Day 55: RBAC Misconfigurations - When Nobody Can Do Anything

THE IDEA:
Deploy overly restrictive RBAC policies and watch ServiceAccounts fail to reconcile,
operators can't create resources, and even admins get locked out of namespaces.

THE SETUP:
Create Roles with missing permissions, bind wrong subjects, and test what breaks
when controllers, pods, and users lack necessary permissions.

WHAT I LEARNED:
- ServiceAccounts need explicit RBAC for API access
- ClusterRole vs Role (cluster-wide vs namespace-scoped)
- Verbs matter: get ≠ list ≠ watch
- Aggregated ClusterRoles combine permissions
- Impersonation requires special permissions

WHY IT MATTERS:
RBAC issues cause:
- Operators stuck unable to reconcile resources
- CI/CD pipelines failing to deploy
- Developers locked out of their namespaces

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-055

Tomorrow: Resource exhaustion and cluster capacity.

---
Chad
Kube Daily - Week 9: Security & Advanced Patterns



---

## Killercoda Lab Instructions



## Step 1: Check default ServiceAccount permissions

```bash
# Create test pod with default ServiceAccount
kubectl run rbac-test --image=nginx

# Try to access API from pod
kubectl exec rbac-test -- sh -c '
  TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
  curl -k -H "Authorization: Bearer $TOKEN" \
    https://kubernetes.default.svc/api/v1/namespaces/default/pods
' 2>&1 || echo "Default SA has no permissions!"
```

Default ServiceAccount has NO permissions!

## Step 2: Create ServiceAccount with limited permissions

```bash
# Create ServiceAccount
kubectl create serviceaccount limited-sa

# Create Role with only 'get' permission
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get"]  # Only get, no list!
EOF

# Bind Role to ServiceAccount
kubectl create rolebinding pod-reader-binding \
  --role=pod-reader \
  --serviceaccount=default:limited-sa
```

## Step 3: Test missing 'list' permission

```bash
# Create pod with limited SA
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: limited-pod
spec:
  serviceAccountName: limited-sa
  containers:
  - name: test
    image: curlimages/curl:latest
    command: ['sleep', '3600']
EOF

# Try to list pods (should fail)
kubectl exec limited-pod -- sh -c '
  TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
  curl -k -H "Authorization: Bearer $TOKEN" \
    https://kubernetes.default.svc/api/v1/namespaces/default/pods
' 2>&1 | grep -i "forbidden"

# Try to get specific pod (should work)
kubectl exec limited-pod -- sh -c '
  TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
  curl -k -H "Authorization: Bearer $TOKEN" \
    https://kubernetes.default.svc/api/v1/namespaces/default/pods/limited-pod
' | jq .metadata.name 2>/dev/null || echo "Get succeeded"
```

## Step 4: Test operator without required permissions

```bash
# Create a simple operator ServiceAccount
kubectl create serviceaccount operator-sa

# Create Role WITHOUT 'create' permission
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: broken-operator-role
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]  # Missing 'create', 'update', 'delete'
EOF

kubectl create rolebinding broken-operator-binding \
  --role=broken-operator-role \
  --serviceaccount=default:operator-sa

# Deploy operator pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: broken-operator
spec:
  serviceAccountName: operator-sa
  containers:
  - name: operator
    image: curlimages/curl:latest
    command: ['sh', '-c']
    args:
    - |
      while true; do
        TOKEN=\$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
        echo "Attempting to create ConfigMap..."
        curl -k -X POST \
          -H "Authorization: Bearer \$TOKEN" \
          -H "Content-Type: application/json" \
          https://kubernetes.default.svc/api/v1/namespaces/default/configmaps \
          -d '{"metadata":{"name":"test-cm"},"data":{"key":"value"}}' 2>&1 | grep -o "forbidden"
        sleep 10
      done
EOF

# Check logs - operator fails to create resources
sleep 15
kubectl logs broken-operator --tail=5
```

## Step 5: Test ClusterRole vs Role scope

```bash
# Create ClusterRole
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
EOF

# Bind ClusterRole to SA (cluster-wide access)
kubectl create clusterrolebinding cluster-reader-binding \
  --clusterrole=cluster-pod-reader \
  --serviceaccount=default:cluster-sa

kubectl create serviceaccount cluster-sa

# Test cross-namespace access
kubectl create namespace other-ns
kubectl run test-pod -n other-ns --image=nginx

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cluster-test
spec:
  serviceAccountName: cluster-sa
  containers:
  - name: test
    image: curlimages/curl:latest
    command: ['sleep', '3600']
EOF

# List pods in other namespace (should work with ClusterRole)
kubectl exec cluster-test -- sh -c '
  TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
  curl -k -H "Authorization: Bearer $TOKEN" \
    https://kubernetes.default.svc/api/v1/namespaces/other-ns/pods
' | jq -r '.items[].metadata.name' 2>/dev/null || echo "ClusterRole allows cross-namespace"
```

## Step 6: Test resourceNames restriction

```bash
# Create Role that only allows access to specific pod
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: specific-pod-only
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "delete"]
  resourceNames: ["rbac-test"]  # Only this pod!
EOF

kubectl create serviceaccount specific-sa
kubectl create rolebinding specific-binding \
  --role=specific-pod-only \
  --serviceaccount=default:specific-sa

# Test access
kubectl auth can-i get pod rbac-test --as=system:serviceaccount:default:specific-sa
kubectl auth can-i get pod limited-pod --as=system:serviceaccount:default:specific-sa
```

## Step 7: Test missing 'watch' permission

```bash
# Create Role without watch
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: no-watch-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]  # No watch!
EOF

kubectl create serviceaccount no-watch-sa
kubectl create rolebinding no-watch-binding \
  --role=no-watch-role \
  --serviceaccount=default:no-watch-sa

# Operators need 'watch' to get real-time updates
kubectl auth can-i watch pods --as=system:serviceaccount:default:no-watch-sa
```

## Step 8: Test aggregated ClusterRoles

```bash
# Create base ClusterRole
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: base-reader
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-reader
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: []  # Rules aggregated from matching ClusterRoles
EOF

# Check aggregated rules
kubectl get clusterrole monitoring-reader -o yaml | grep -A 20 "rules:"
```

## Step 9: Test subresource permissions

```bash
# Create Role with logs access
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: logs-reader
rules:
- apiGroups: [""]
  resources: ["pods/log"]  # Subresource!
  verbs: ["get"]
EOF

kubectl create serviceaccount logs-sa
kubectl create rolebinding logs-binding \
  --role=logs-reader \
  --serviceaccount=default:logs-sa

# Test - can read logs but not pod details
kubectl auth can-i get pods/log --as=system:serviceaccount:default:logs-sa
kubectl auth can-i get pods --as=system:serviceaccount:default:logs-sa
```

## Step 10: Test impersonation permissions

```bash
# Create Role that allows impersonation
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: impersonator
rules:
- apiGroups: [""]
  resources: ["users", "groups", "serviceaccounts"]
  verbs: ["impersonate"]
EOF

kubectl create serviceaccount impersonator-sa
kubectl create clusterrolebinding impersonator-binding \
  --clusterrole=impersonator \
  --serviceaccount=default:impersonator-sa

# Test impersonation
kubectl auth can-i impersonate users --as=system:serviceaccount:default:impersonator-sa
```

## Step 11: Test wildcard permissions

```bash
# Create Role with wildcards (dangerous!)
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: wildcard-role
rules:
- apiGroups: ["*"]  # All API groups
  resources: ["*"]  # All resources
  verbs: ["*"]      # All verbs
EOF

kubectl create serviceaccount wildcard-sa
kubectl create rolebinding wildcard-binding \
  --role=wildcard-role \
  --serviceaccount=default:wildcard-sa

# This SA can do anything in the namespace!
kubectl auth can-i delete secrets --as=system:serviceaccount:default:wildcard-sa
```

## Step 12: Test custom resource permissions

```bash
# Create CRD
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
  names:
    kind: Database
    plural: databases
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
            properties:
              engine:
                type: string
EOF

# Create Role for custom resource
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: database-operator
rules:
- apiGroups: ["example.com"]
  resources: ["databases"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["example.com"]
  resources: ["databases/status"]  # Status subresource
  verbs: ["get", "update", "patch"]
EOF

kubectl create serviceaccount db-operator-sa
kubectl create rolebinding db-operator-binding \
  --role=database-operator \
  --serviceaccount=default:db-operator-sa
```

## Step 13: Debug RBAC issues

```bash
# Check if SA can perform action
kubectl auth can-i create deployments --as=system:serviceaccount:default:limited-sa

# List all permissions for SA
kubectl auth can-i --list --as=system:serviceaccount:default:limited-sa

# Get RoleBindings for SA
kubectl get rolebindings -o json | jq -r '
  .items[] |
  select(.subjects[]?.name == "limited-sa") |
  "\(.metadata.name): \(.roleRef.name)"
'

# Check ClusterRoleBindings
kubectl get clusterrolebindings -o json | jq -r '
  .items[] |
  select(.subjects[]?.name == "limited-sa") |
  "\(.metadata.name): \(.roleRef.name)"
'
```

## Step 14: Test RBAC for system components

```bash
# Check kube-system ServiceAccounts
kubectl get sa -n kube-system

# Check permissions for core-dns
kubectl auth can-i list services --as=system:serviceaccount:kube-system:coredns -n kube-system

# Check metrics-server permissions
kubectl auth can-i get nodes --as=system:serviceaccount:kube-system:metrics-server 2>/dev/null || echo "metrics-server SA permissions"
```

## Step 15: Create least-privilege role

```bash
# Example: CI/CD deployer role
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: default
rules:
# Deployments
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: ["apps"]
  resources: ["deployments/status"]
  verbs: ["get"]
# Services
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "create", "update", "patch"]
# ConfigMaps (read-only)
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
# Secrets (read-only)
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
# Events (for logs)
- apiGroups: [""]
  resources: ["events"]
  verbs: ["get", "list"]
# Pod logs
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
EOF

kubectl create serviceaccount deployer-sa
kubectl create rolebinding deployer-binding \
  --role=deployer \
  --serviceaccount=default:deployer-sa

# Verify permissions
echo "Deployer can:"
kubectl auth can-i create deployments --as=system:serviceaccount:default:deployer-sa && echo "  ✓ Create deployments"
kubectl auth can-i delete secrets --as=system:serviceaccount:default:deployer-sa || echo "  ✗ Delete secrets (good!)"
```

## Key Observations

✅ **get ≠ list ≠ watch** - each verb is separate permission
✅ **ClusterRole** - cluster-wide, Role is namespace-scoped
✅ **Default SA** - has no permissions by default
✅ **Subresources** - require separate permissions (pods/log, pods/status)
✅ **resourceNames** - restricts to specific resource instances
✅ **Aggregation** - combines multiple ClusterRoles

## Production Patterns

**Operator ServiceAccount:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-operator
  namespace: myapp-system
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
# Dependent resources
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# Events
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "patch"]
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
  namespace: myapp-system
```

**Read-only developer access:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-readonly
  namespace: development
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log", "pods/portforward"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]  # Allow kubectl exec for debugging
```

**CI/CD ServiceAccount:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cicd-deployer
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cicd-deployer
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["services", "configmaps"]
  verbs: ["get", "list", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]  # Read-only secrets
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["get", "list", "create"]
```

**RBAC audit script:**
```bash
#!/bin/bash
# rbac-audit.sh

echo "=== RBAC Audit Report ==="

echo -e "\n1. ServiceAccounts with cluster-admin:"
kubectl get clusterrolebindings -o json | jq -r '
  .items[] |
  select(.roleRef.name == "cluster-admin") |
  .subjects[]? |
  select(.kind == "ServiceAccount") |
  "\(.namespace)/\(.name)"
'

echo -e "\n2. Roles with wildcard permissions:"
kubectl get roles,clusterroles -A -o json | jq -r '
  .items[] |
  select(.rules[]? | .verbs[]? == "*") |
  "\(.metadata.namespace // "cluster")/\(.metadata.name)"
'

echo -e "\n3. ServiceAccounts with no bindings:"
for sa in $(kubectl get sa -A -o json | jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name)"'); do
  ns=$(echo $sa | cut -d/ -f1)
  name=$(echo $sa | cut -d/ -f2)

  bindings=$(kubectl get rolebindings,clusterrolebindings -A -o json | jq -r "
    .items[] |
    select(.subjects[]?.kind == \"ServiceAccount\" and .subjects[]?.name == \"$name\") |
    .metadata.name
  ")

  if [ -z "$bindings" ]; then
    echo "  $sa (unused)"
  fi
done

echo -e "\n4. Overly permissive bindings:"
kubectl get clusterrolebindings -o json | jq -r '
  .items[] |
  select(.subjects[]?.kind == "Group" and .subjects[]?.name == "system:authenticated") |
  "\(.metadata.name): \(.roleRef.name)"
'
```

## Cleanup

```bash
kubectl delete pod rbac-test limited-pod broken-operator cluster-test
kubectl delete sa limited-sa operator-sa cluster-sa specific-sa no-watch-sa logs-sa impersonator-sa wildcard-sa db-operator-sa deployer-sa
kubectl delete role pod-reader broken-operator-role specific-pod-only no-watch-role logs-reader wildcard-role database-operator deployer
kubectl delete rolebinding pod-reader-binding broken-operator-binding specific-binding no-watch-binding logs-binding wildcard-binding db-operator-binding deployer-binding
kubectl delete clusterrole cluster-pod-reader base-reader monitoring-reader impersonator
kubectl delete clusterrolebinding cluster-reader-binding impersonator-binding
kubectl delete crd databases.example.com
kubectl delete namespace other-ns
```

---
Next: Day 56 - Resource Exhaustion and Cluster Capacity
