# Day 16: RBAC Lockout - Permission Denied Debugging

## THE IDEA:
Create a ServiceAccount with minimal RBAC permissions and watch kubectl 
commands fail mysteriously. Learn to decode "User cannot X" errors and 
understand the difference between Roles and ClusterRoles.

## THE SETUP:
Deploy a pod using a restricted ServiceAccount that tries to list other pods. 
Observe the permission denied error and gradually grant access.

## WHAT I LEARNED:
- Default ServiceAccount has almost no permissions
- Role is namespace-scoped, ClusterRole is cluster-wide
- RoleBinding can reference ClusterRole (namespace-scoped use of cluster role)
- "system:serviceaccount:namespace:name" is the full identity format

## WHY IT MATTERS:
RBAC issues cause:
- Operators failing silently
- CI/CD unable to deploy
- Monitoring agents unable to scrape metrics

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-016

Tomorrow: Admission webhooks that timeout and block all deployments.

---

## Killercoda Lab Instructions

### Step 1: Create a restricted ServiceAccount

```bash
kubectl create namespace rbac-test
kubectl create serviceaccount restricted-sa -n rbac-test

```

### Step 2: Deploy a pod using this ServiceAccount (with kubectl)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: kubectl-pod
  namespace: rbac-test
spec:
  serviceAccountName: restricted-sa
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ['sleep', '3600']
EOF

```

### Step 3: Try to list pods (fails!)

```bash
kubectl exec -n rbac-test kubectl-pod -- kubectl get pods

```

**Error:** "Error from server (Forbidden): pods is forbidden: User
"system:serviceaccount:rbac-test:restricted-sa" cannot list resource "pods""

### Step 4: Check what permissions the SA has

```bash
kubectl auth can-i list pods --as=system:serviceaccount:rbac-test:restricted-sa -n rbac-test

```

Output: `no`

```bash
kubectl auth can-i --list --as=system:serviceaccount:rbac-test:restricted-sa -n rbac-test

```

Shows only self-subject access review (minimal permissions).

### Step 5: Create a Role with pod read permissions

```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: rbac-test
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
EOF

```

**Role alone doesn't grant access - need RoleBinding!**

### Step 6: Bind the Role to the ServiceAccount

```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: rbac-test
subjects:
- kind: ServiceAccount
  name: restricted-sa
  namespace: rbac-test
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF

```

### Step 7: Test again (success!)

```bash
kubectl exec -n rbac-test kubectl-pod -- kubectl get pods -n rbac-test

```

Now it works! ServiceAccount can list pods.

**Try to list pods in different namespace:**

```bash
kubectl exec -n rbac-test kubectl-pod -- kubectl get pods -n default

```

Still fails - Role is namespace-scoped!

### Step 8: Test other operations (still denied)

```bash
# Can we delete pods?
kubectl exec -n rbac-test kubectl-pod -- kubectl delete pod kubectl-pod -n rbac-test

```

Error: "delete" verb not granted.

```bash
# Can we list services?
kubectl exec -n rbac-test kubectl-pod -- kubectl get services -n rbac-test

```

Error: "services" resource not granted.

### Step 9: Create ClusterRole for cluster-wide access

```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader-cluster
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get", "list"]
EOF

```

### Step 10: ClusterRoleBinding for cluster-wide access

```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-pods-global
subjects:
- kind: ServiceAccount
  name: restricted-sa
  namespace: rbac-test
roleRef:
  kind: ClusterRole
  name: pod-reader-cluster
  apiGroup: rbac.authorization.k8s.io
EOF

```

**Test cross-namespace access:**

```bash
kubectl exec -n rbac-test kubectl-pod -- kubectl get pods --all-namespaces

```

Works! ClusterRole + ClusterRoleBinding = cluster-wide access.

### Step 11: RoleBinding with ClusterRole (namespace-scoped use)

```bash
# Create new namespace and ServiceAccount
kubectl create namespace app-team
kubectl create serviceaccount app-sa -n app-team

# Bind ClusterRole with RoleBinding (limits to namespace!)
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: app-team
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: app-team
roleRef:
  kind: ClusterRole
  name: view  # Built-in ClusterRole
  apiGroup: rbac.authorization.k8s.io
EOF

```

**This allows reusing ClusterRoles in namespace-scoped way.**

### Step 12: Test aggregated ClusterRoles

```bash
# Check what 'view' ClusterRole includes
kubectl describe clusterrole view | head -30

# Check what 'edit' ClusterRole includes
kubectl describe clusterrole edit | head -30

# Check what 'admin' ClusterRole includes
kubectl describe clusterrole admin | head -30

```

These are aggregated roles that combine multiple permissions.

### Step 13: Create custom aggregated ClusterRole

```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-reader
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
rules:
- apiGroups: [""]
  resources: ["pods", "services", "endpoints"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: []  # Rules auto-populated by aggregation
EOF

```

**Check aggregated result:**

```bash
kubectl describe clusterrole monitoring

```

Rules are pulled from all ClusterRoles with matching label!

### Step 14: Debug RBAC with auth can-i

```bash
# Check specific permission
kubectl auth can-i create deployments --as=system:serviceaccount:rbac-test:restricted-sa -n rbac-test

# Check all permissions for a user
kubectl auth can-i --list --as=system:serviceaccount:rbac-test:restricted-sa -n rbac-test

# Check from inside pod
kubectl exec -n rbac-test kubectl-pod -- kubectl auth can-i create pods -n rbac-test

```

### Step 15: Test resource and verb granularity

```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deploy-manager
  namespace: rbac-test
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: ["apps"]
  resources: ["deployments/scale"]
  verbs: ["update", "patch"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
EOF

```

**Subresources** (like `pods/log`, `deployments/scale`) require explicit permission!

### Step 16: Test resource names restriction

```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: specific-secret-reader
  namespace: rbac-test
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["my-secret"]  # Only this specific secret!
  verbs: ["get"]
EOF

```

Grant access to ONE specific resource by name.

### Key Observations

✅ **Default SA has minimal permissions** - explicitly grant what's needed
✅ **Role vs ClusterRole** - namespace-scoped vs cluster-scoped
✅ **RoleBinding + ClusterRole** - namespace-scoped use of cluster role
✅ **Subresources** - require explicit permission (pods/log, deployments/scale)
✅ **auth can-i** - essential debugging tool

### Production Patterns

**CI/CD ServiceAccount:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployer
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["services", "configmaps", "secrets"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]

```

**Monitoring ServiceAccount:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources: ["nodes", "nodes/metrics", "services", "endpoints", "pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]
- nonResourceURLs: ["/metrics", "/metrics/cadvisor"]
  verbs: ["get"]

```

**App with minimum permissions:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-minimal
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["app-config"]
  verbs: ["get"]
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["app-secrets"]
  verbs: ["get"]

```

### Cleanup

```bash
kubectl delete namespace rbac-test app-team
kubectl delete clusterrole pod-reader-cluster monitoring monitoring-reader 2>/dev/null
kubectl delete clusterrolebinding read-pods-global 2>/dev/null

```

---

Next: Day 17 - Admission Webhook Timeouts