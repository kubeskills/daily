## Day 57: Pod Security Standards - Enforcement Failures

### Email Copy

**Subject:** Day 57: Pod Security Standards - When Policies Block Everything

THE IDEA:
Enable Pod Security Standards in restricted mode and watch deployments fail due to
privileged containers, host namespaces, and dangerous capabilities. Learn what breaks.

THE SETUP:
Apply baseline and restricted policies to namespaces, deploy workloads that violate
them, and discover which security requirements block common patterns.

WHAT I LEARNED:
- Three profiles: privileged (unrestricted), baseline (minimal), restricted (hardened)
- Restricted requires: runAsNonRoot, drop ALL capabilities, read-only root FS
- Enforcement modes: enforce (block), warn (allow + warning), audit (log only)
- Namespace labels control which profile applies
- Some legitimate workloads need baseline (not restricted)

WHY IT MATTERS:
Pod Security issues cause:
- Deployments blocked in production (pods never start)
- CI/CD pipelines failing on security violations
- Existing workloads breaking after cluster upgrade

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-057

Tomorrow: Ingress failures and traffic routing issues.

---
Chad



---

## Killercoda Lab Instructions




## Step 1: Check Pod Security Admission status

```bash
# Check if PSA is enabled (default in K8s 1.23+)
kubectl api-versions | grep admissionregistration

# Check current namespace labels
kubectl get namespaces default -o yaml | grep pod-security
```

## Step 2: Create test namespace with privileged profile (no restrictions)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: privileged-ns
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
EOF
```

## Step 3: Deploy privileged pod (works in privileged-ns)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
  namespace: privileged-ns
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      privileged: true
      runAsUser: 0
      capabilities:
        add: ["SYS_ADMIN", "NET_ADMIN"]
EOF

# Check status
kubectl get pod privileged-pod -n privileged-ns
```

Works! Privileged profile allows everything.

## Step 4: Create namespace with baseline profile

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: baseline-ns
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: baseline
    pod-security.kubernetes.io/warn: baseline
EOF
```

## Step 5: Test baseline violations

```bash
# Try privileged container (BLOCKED by baseline)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: blocked-privileged
  namespace: baseline-ns
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      privileged: true
EOF
```

**Error:** "pods 'blocked-privileged' is forbidden: violates PodSecurity 'baseline:latest': privileged"

## Step 6: Test host namespace violations

```bash
# Try hostNetwork (BLOCKED by baseline)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: blocked-hostnetwork
  namespace: baseline-ns
spec:
  hostNetwork: true
  containers:
  - name: app
    image: nginx
EOF
```

**Error:** "violates PodSecurity 'baseline:latest': host namespaces (hostNetwork=true)"

## Step 7: Test hostPath violations

```bash
# Try hostPath (BLOCKED by baseline)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: blocked-hostpath
  namespace: baseline-ns
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: host
      mountPath: /host
  volumes:
  - name: host
    hostPath:
      path: /
      type: Directory
EOF
```

**Error:** "violates PodSecurity 'baseline:latest': hostPath volumes"

## Step 8: Deploy baseline-compliant pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: baseline-compliant
  namespace: baseline-ns
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
EOF
```

Works! Meets baseline requirements.

## Step 9: Create namespace with restricted profile

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: restricted-ns
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
EOF
```

## Step 10: Test restricted violations - running as root

```bash
# Try running as root (BLOCKED by restricted)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: blocked-root
  namespace: restricted-ns
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
EOF
```

**Error:** "violates PodSecurity 'restricted:latest': runAsNonRoot != true"

## Step 11: Test restricted violations - missing seccomp

```bash
# Try without seccomp profile (BLOCKED)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: blocked-seccomp
  namespace: restricted-ns
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
  - name: app
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
EOF
```

**Error:** "violates PodSecurity 'restricted:latest': seccompProfile"

## Step 12: Deploy fully restricted-compliant pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: restricted-compliant
  namespace: restricted-ns
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:latest
    securityContext:
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop: ["ALL"]
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /var/run
  volumes:
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
EOF

# Check status
kubectl get pod restricted-compliant -n restricted-ns
```

Works! Fully compliant with restricted profile.

## Step 13: Test warn mode (allows but warns)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: warn-ns
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
EOF

# Deploy non-compliant pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: warned-pod
  namespace: warn-ns
spec:
  containers:
  - name: app
    image: nginx
EOF
```

**Warning shown but pod allowed!**

## Step 14: Test audit mode (logs violations)

```bash
# Check audit logs (if accessible)
kubectl get events -n warn-ns --sort-by='.lastTimestamp' | grep -i "audit"

# Or check API server logs
# kubectl logs -n kube-system -l component=kube-apiserver --tail=50 | grep audit
```

## Step 15: Test Deployment with security violations

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-test
  namespace: restricted-ns
spec:
  replicas: 3
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
EOF
```

**Error:** Deployment created but pods fail to start (violates restricted)

```bash
kubectl get deployment deployment-test -n restricted-ns
kubectl get events -n restricted-ns --sort-by='.lastTimestamp' | grep -i "forbidden"
```

## Step 16: Check which fields violate restricted

```bash
# List all restricted requirements
cat << 'EOF'
Restricted Profile Requirements:
1. runAsNonRoot: true
2. allowPrivilegeEscalation: false
3. capabilities.drop: ["ALL"]
4. seccompProfile.type: RuntimeDefault or Localhost
5. No privileged containers
6. No host namespaces (hostNetwork, hostPID, hostIPC)
7. No hostPath volumes
8. Volume types limited (no hostPath, gcePersistentDisk, etc)
9. AppArmor profile (runtime/default)
10. SELinux must be unset or specific values
EOF
```

## Step 17: Test common workload patterns

```bash
# Pattern 1: Database (needs fsGroup)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: database-pod
  namespace: restricted-ns
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 999
    fsGroup: 999
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: db
    image: postgres:15
    env:
    - name: POSTGRES_PASSWORD
      value: password
    securityContext:
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      runAsUser: 999
      capabilities:
        drop: ["ALL"]
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: data
    emptyDir: {}
EOF

kubectl get pod database-pod -n restricted-ns
```

## Step 18: Test StatefulSet with restricted

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: restricted-statefulset
  namespace: restricted-ns
spec:
  serviceName: restricted-svc
  replicas: 2
  selector:
    matchLabels:
      app: restricted-app
  template:
    metadata:
      labels:
        app: restricted-app
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: app
        image: nginx:latest
        securityContext:
          allowPrivilegeEscalation: false
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop: ["ALL"]
          readOnlyRootFilesystem: true
        volumeMounts:
        - name: cache
          mountPath: /var/cache/nginx
        - name: run
          mountPath: /var/run
      volumes:
      - name: cache
        emptyDir: {}
      - name: run
        emptyDir: {}
EOF

kubectl get statefulset restricted-statefulset -n restricted-ns
kubectl get pods -n restricted-ns -l app=restricted-app
```

## Step 19: Test DaemonSet (often needs baseline not restricted)

```bash
# DaemonSets often need hostPath, hostNetwork
# Use baseline profile for system DaemonSets
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: system-ns
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: baseline
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
  namespace: system-ns
spec:
  selector:
    matchLabels:
      app: logs
  template:
    metadata:
      labels:
        app: logs
    spec:
      containers:
      - name: collector
        image: busybox
        command: ['sh', '-c', 'tail -f /var/log/syslog']
        securityContext:
          allowPrivilegeEscalation: false
        volumeMounts:
        - name: logs
          mountPath: /var/log
          readOnly: true
      volumes:
      - name: logs
        hostPath:
          path: /var/log
          type: Directory
EOF

kubectl get daemonset log-collector -n system-ns
```

Works in baseline (hostPath allowed)!

## Step 20: Check exemptions (if configured)

```bash
# Exemptions allow specific users/namespaces/images to bypass PSA
# Configured at API server level with --admission-control-config-file

cat << 'EOF'
Example exemption config:
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: "baseline"
      audit: "restricted"
      warn: "restricted"
    exemptions:
      usernames: []
      runtimeClasses: []
      namespaces: ["kube-system"]
EOF
```

## Key Observations

✅ **Three profiles** - privileged, baseline, restricted
✅ **Restricted hardest** - requires runAsNonRoot, drop ALL caps, seccomp
✅ **Three modes** - enforce (block), warn (allow+warn), audit (log)
✅ **Namespace labels** - control which profile applies
✅ **Common violations** - root user, privileged, hostPath, hostNetwork
✅ **Some workloads** - legitimately need baseline (not restricted)

## Production Patterns

**Default namespace security:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce restricted by default
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    # Warn on baseline violations
    pod-security.kubernetes.io/warn: baseline
    pod-security.kubernetes.io/warn-version: latest
    # Audit everything
    pod-security.kubernetes.io/audit: baseline
    pod-security.kubernetes.io/audit-version: latest
```

**Restricted-compliant Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: secure-app
  template:
    metadata:
      labels:
        app: secure-app
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: app
        image: myapp:v1.0
        securityContext:
          allowPrivilegeEscalation: false
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop: ["ALL"]
          readOnlyRootFilesystem: true
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
```

**Baseline for system workloads:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    # Baseline for monitoring tools that need hostPath
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: baseline
    pod-security.kubernetes.io/audit: restricted
```

**Migration strategy:**
```bash
# Step 1: Audit mode first (no enforcement)
kubectl label namespace production \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

# Step 2: Review violations
kubectl get events -n production | grep "violates PodSecurity"

# Step 3: Fix workloads
# Update deployments to be compliant

# Step 4: Enforce
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted --overwrite
```

**Check compliance script:**
```bash
#!/bin/bash
# check-pod-security.sh

echo "=== Pod Security Compliance Check ==="

for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  echo -e "\nNamespace: $ns"

  # Get PSA labels
  enforce=$(kubectl get ns $ns -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/enforce}' 2>/dev/null || echo "none")
  warn=$(kubectl get ns $ns -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/warn}' 2>/dev/null || echo "none")

  echo "  Enforce: $enforce"
  echo "  Warn: $warn"

  # Count pods
  pod_count=$(kubectl get pods -n $ns --no-headers 2>/dev/null | wc -l)
  echo "  Pods: $pod_count"
done

echo -e "\n=== Recommendations ==="
echo "1. Set enforce=restricted for application namespaces"
echo "2. Use enforce=baseline for system namespaces"
echo "3. Always enable audit=restricted for visibility"
```

## Cleanup

```bash
kubectl delete namespace privileged-ns baseline-ns restricted-ns warn-ns system-ns
```

---
Next: Day 58 - Ingress Failures and Traffic Routing
