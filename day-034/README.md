## Day 34: Multi-Tenancy - Namespace Isolation Breakdown

## THE IDEA:
Create namespaces for Team A and Team B with ResourceQuotas and NetworkPolicies. 
Watch isolation break when a pod in Team A accidentally accesses Team B's services.

## THE SETUP:
Deploy multi-tenant setup with RBAC, quotas, and network policies. Test cross-
namespace access, quota violations, and privilege escalation attempts.

## WHAT I LEARNED:
- Namespaces provide soft isolation (not security boundary alone)
- NetworkPolicies can block cross-namespace traffic
- RBAC prevents unauthorized namespace access
- ResourceQuotas prevent resource monopolization
- Service DNS works across namespaces (team-b.namespace-b.svc.cluster.local)

## WHY IT MATTERS:
Multi-tenancy failures cause:
- Data breaches between tenants
- Resource starvation from noisy neighbors
- Privilege escalation across teams

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/playgrounds/scenario/kubernetes


---

# Killercoda Lab Instructions


## Step 1: Create tenant namespaces

```bash
kubectl create namespace team-a
kubectl create namespace team-b
kubectl create namespace team-c
```

## Step 2: Set ResourceQuotas per namespace

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "2"
    requests.memory: "4Gi"
    limits.cpu: "4"
    limits.memory: "8Gi"
    pods: "10"
    services: "5"
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-b
spec:
  hard:
    requests.cpu: "1"
    requests.memory: "2Gi"
    limits.cpu: "2"
    limits.memory: "4Gi"
    pods: "5"
    services: "3"
EOF
```

## Step 3: Create ServiceAccounts and RBAC per team

```bash
# Team A - admin in their namespace
kubectl create serviceaccount team-a-admin -n team-a

cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: team-admin
  namespace: team-a
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-admin-binding
  namespace: team-a
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: team-admin
subjects:
- kind: ServiceAccount
  name: team-a-admin
  namespace: team-a
EOF
```

## Step 4: Deploy application in Team A

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-a
  namespace: team-a
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app-a
      team: a
  template:
    metadata:
      labels:
        app: app-a
        team: a
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: app-a-svc
  namespace: team-a
spec:
  selector:
    app: app-a
  ports:
  - port: 80
    targetPort: 80
EOF
```

## Step 5: Deploy application in Team B

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-b
  namespace: team-b
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-b
      team: b
  template:
    metadata:
      labels:
        app: app-b
        team: b
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: app-b-svc
  namespace: team-b
spec:
  selector:
    app: app-b
  ports:
  - port: 80
    targetPort: 80
EOF
```

## Step 6: Test cross-namespace access (allowed by default!)

```bash
kubectl run -it --rm test -n team-a --image=curlimages/curl -- curl -m 5 http://app-b-svc.team-b.svc.cluster.local
```

**Success!** Team A can access Team B's service.

## Step 7: Apply NetworkPolicy to isolate Team B

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-other-namespaces
  namespace: team-b
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}  # Only from same namespace
  egress:
  - to:
    - podSelector: {}  # Only to same namespace
  - to:  # Allow DNS
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
EOF
```

## Step 8: Test cross-namespace access again (blocked!)

```bash
kubectl run -it --rm test -n team-a --image=curlimages/curl -- curl -m 5 http://app-b-svc.team-b.svc.cluster.local 2>&1 || echo "Blocked!"
```

Timeout! NetworkPolicy blocked it.

## Step 9: Test from within Team B (allowed)

```bash
kubectl run -it --rm test -n team-b --image=curlimages/curl -- curl -m 5 http://app-b-svc
```

Works! Same namespace allowed.

## Step 10: Test ResourceQuota violation

```bash
# Try to exceed Team B's pod quota (limit: 5)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quota-buster
  namespace: team-b
spec:
  replicas: 10  # Exceeds quota!
  selector:
    matchLabels:
      app: buster
  template:
    metadata:
      labels:
        app: buster
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
EOF
```

**Check pods:**
```bash
kubectl get deployment quota-buster -n team-b
kubectl get replicaset -n team-b
```

Only 5 pods created! Quota enforced.

## Step 11: Test RBAC isolation

```bash
# Try to list pods in team-b as team-a-admin
kubectl auth can-i list pods --namespace=team-b --as=system:serviceaccount:team-a:team-a-admin
```

Output: `no`

**Try anyway:**
```bash
TOKEN=$(kubectl create token team-a-admin -n team-a --duration=1h)
kubectl --token=$TOKEN get pods -n team-b 2>&1 | grep -i forbidden
```

Forbidden! RBAC blocked it.

## Step 12: Test privileged pod in isolated namespace

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: privileged-attempt
  namespace: team-a
spec:
  containers:
  - name: bad
    image: nginx
    securityContext:
      privileged: true
EOF
```

**Add PSA to block:**
```bash
kubectl label namespace team-a pod-security.kubernetes.io/enforce=baseline
kubectl delete pod privileged-attempt -n team-a
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: privileged-attempt
  namespace: team-a
spec:
  containers:
  - name: bad
    image: nginx
    securityContext:
      privileged: true
EOF
```

Blocked by Pod Security Admission!

## Step 13: Test LimitRange for default limits

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-limits
  namespace: team-a
spec:
  limits:
  - max:
      cpu: "1"
      memory: "1Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
    default:
      cpu: "200m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    type: Container
EOF
```

**Deploy pod without resources:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: auto-limits
  namespace: team-a
spec:
  containers:
  - name: app
    image: nginx
EOF
```

**Check assigned resources:**
```bash
kubectl get pod auto-limits -n team-a -o jsonpath='{.spec.containers[0].resources}'
```

Defaults applied!

## Step 14: Test namespace-scoped secrets

```bash
# Create secret in Team A
kubectl create secret generic team-a-secret -n team-a --from-literal=key=valueA

# Try to access from Team B
kubectl run -it --rm test -n team-b --image=busybox -- sh -c 'ls /var/run/secrets/kubernetes.io/serviceaccount/'
```

Secrets are namespace-scoped - Team B can't access Team A's secrets.

## Step 15: Test hierarchical namespaces (conceptual)

```bash
# Hierarchical Namespace Controller (HNC) example
# Team A parent, Team A-Dev child

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: team-a-dev
  labels:
    team: a
    environment: dev
EOF

# Apply same ResourceQuota to child
kubectl apply -n team-a-dev -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
spec:
  hard:
    requests.cpu: "500m"
    requests.memory: "1Gi"
EOF
```

## Key Observations

✅ **Namespaces** - logical isolation, not security boundary alone
✅ **NetworkPolicy** - blocks cross-namespace traffic
✅ **RBAC** - prevents unauthorized access
✅ **ResourceQuota** - prevents resource monopolization
✅ **LimitRange** - sets default resource constraints
✅ **PSA** - enforces security standards per namespace

## Production Patterns

**Tenant namespace template:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-{{.TenantName}}
  labels:
    tenant: {{.TenantName}}
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-quota
  namespace: tenant-{{.TenantName}}
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "50"
    services: "20"
    persistentvolumeclaims: "10"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-limits
  namespace: tenant-{{.TenantName}}
spec:
  limits:
  - max:
      cpu: "2"
      memory: "4Gi"
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    type: Container
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace
  namespace: tenant-{{.TenantName}}
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}
  egress:
  - to:
    - podSelector: {}
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

## Cleanup

```bash
kubectl delete namespace team-a team-b team-c team-a-dev
```

---
