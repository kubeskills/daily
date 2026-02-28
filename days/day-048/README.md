## Day 48: Cluster Upgrades - Version Skew Chaos

### Email Copy

**Subject:** Day 48: Cluster Upgrades - When Version Skew Breaks Everything

THE IDEA:
Simulate a cluster upgrade gone wrong where control plane and nodes are at
incompatible versions. Watch API calls fail and pods get stuck due to version skew.

THE SETUP:
Check Kubernetes version compatibility rules, identify version mismatches, and
test deprecated API versions that break after upgrade.

WHAT I LEARNED:
- Control plane can be N+1 ahead of nodes (but not reverse)
- kubectl can be ±1 version from API server
- API versions deprecate (v1beta1 → v1)
- CRDs may need upgrade before cluster upgrade
- Rolling node upgrades minimize downtime

WHY IT MATTERS:
Version skew issues cause:
- API calls rejected after upgrade
- Pods failing to schedule on new nodes
- Custom resources becoming invalid
- Rollback complexity and risk

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-048

Tomorrow: Backup and restore failures with Velero.

---
Chad
Kube Daily - Week 8: Disaster Recovery & Operations


---

## Killercoda Lab Instructions

## Step 1: Check current cluster version

```bash
kubectl version --short

# Check server version
kubectl version -o json | jq -r '.serverVersion.gitVersion'

# Check node versions
kubectl get nodes -o wide
```

## Step 2: Check API server supported versions

```bash
kubectl api-versions | sort
```

## Step 3: Test deprecated API versions

```bash
# Check for deprecated APIs in current resources
kubectl get deployments.v1.apps -A
kubectl get ingresses.v1.networking.k8s.io -A

# Older clusters might have:
# kubectl get deployments.v1beta1.apps (deprecated)
# kubectl get ingresses.v1beta1.networking.k8s.io (deprecated)
```

## Step 4: Create resource with deprecated API (simulation)

```bash
# Example of API version that might be deprecated
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-version-test
  annotations:
    deprecated-by: "apps/v1"
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
      - name: nginx
        image: nginx
EOF
```

**Check which API version was stored:**
```bash
kubectl get deployment api-version-test -o yaml | grep "apiVersion:"
```

## Step 5: Test version compatibility matrix

```bash
# Kubernetes version skew policy:
# - kube-apiserver: N
# - kubelet: N-2 to N
# - kube-controller-manager: N-1 to N
# - kube-scheduler: N-1 to N
# - kubectl: N-1 to N+1

# Check component versions
kubectl get nodes -o json | jq -r '.items[] | "\(.metadata.name): \(.status.nodeInfo.kubeletVersion)"'
```

## Step 6: Identify deprecated APIs in manifests

```bash
# Install kubectl-convert plugin (if available)
# kubectl convert -f old-manifest.yaml --output-version apps/v1

# Check for deprecated APIs
cat > /tmp/check-deprecated.sh << 'EOF'
#!/bin/bash
echo "Checking for potentially deprecated APIs..."

# Common deprecated APIs
deprecated_apis=(
  "extensions/v1beta1"
  "apps/v1beta1"
  "apps/v1beta2"
  "networking.k8s.io/v1beta1"
  "policy/v1beta1"
  "rbac.authorization.k8s.io/v1beta1"
)

for api in "${deprecated_apis[@]}"; do
  echo "Checking for $api..."
  kubectl get --raw /apis 2>/dev/null | grep -q "$api" && echo "  Found: $api" || echo "  Not present"
done
EOF

chmod +x /tmp/check-deprecated.sh
/tmp/check-deprecated.sh
```

## Step 7: Test CRD version compatibility

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
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
              cronSpec:
                type: string
              image:
                type: string
  - name: v1beta1
    served: true
    storage: false
    deprecated: true
    deprecationWarning: "v1beta1 is deprecated, use v1"
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              cronSpec:
                type: string
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
EOF
```

**Test deprecated version:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: stable.example.com/v1beta1
kind: CronTab
metadata:
  name: test-crontab
spec:
  cronSpec: "* * * * */5"
EOF
```

**Check warning:**
```bash
kubectl get crontabs.stable.example.com -o yaml
```

## Step 8: Simulate upgrade pre-check

```bash
# Check what would break in upgrade
cat > /tmp/upgrade-precheck.sh << 'EOF'
#!/bin/bash
echo "=== Upgrade Pre-Check ==="

echo -e "\n1. Checking for deprecated APIs in use..."
kubectl get all -A -o json | jq -r '.items[] | "\(.apiVersion) - \(.kind)"' | sort -u

echo -e "\n2. Checking node versions..."
kubectl get nodes -o custom-columns=NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion

echo -e "\n3. Checking for beta resources..."
kubectl api-resources | grep beta

echo -e "\n4. Checking admission webhooks..."
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations -A

echo -e "\n5. Checking PodSecurityPolicy (deprecated in 1.21, removed in 1.25)..."
kubectl get psp 2>/dev/null || echo "  PSP not found (good if cluster is 1.25+)"

echo -e "\n6. Checking for deprecated Ingress API..."
kubectl get ingress -A -o json 2>/dev/null | jq -r '.items[] | "\(.apiVersion) - \(.metadata.namespace)/\(.metadata.name)"'

echo -e "\nPre-check complete!"
EOF

chmod +x /tmp/upgrade-precheck.sh
/tmp/upgrade-precheck.sh
```

## Step 9: Test kubectl version skew

```bash
# Check current kubectl version
kubectl version --client --short

# Simulate version mismatch (conceptual)
# If kubectl is 1.28 and server is 1.30, should still work (within ±1)
# If kubectl is 1.26 and server is 1.30, may have issues

echo "kubectl compatibility: server ±1 minor version"
```

## Step 10: Check for removed feature gates

```bash
# Feature gates can be removed in new versions
# Check current feature gates
kubectl get --raw /metrics | grep feature_enabled || echo "Metrics not available"

# Common removed features:
# - PodSecurityPolicy (removed in 1.25)
# - EphemeralContainers (graduated to GA in 1.25)
# - CronJobControllerV2 (graduated in 1.21)
```

## Step 11: Test backward compatibility

```bash
# Create resource with old API version
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: legacy-service
spec:
  selector:
    app: legacy
  ports:
  - port: 80
EOF

# Read with explicit version
kubectl get service legacy-service -o yaml | grep apiVersion
```

## Step 12: Check for alpha APIs (unstable)

```bash
# Alpha APIs might be removed between versions
kubectl api-resources | grep -E "alpha|v[0-9]+alpha[0-9]+"

# Check if any resources use alpha APIs
kubectl get all -A -o json | jq -r '.items[] | select(.apiVersion | contains("alpha")) | "\(.kind): \(.apiVersion)"' 2>/dev/null || echo "No alpha APIs in use"
```

## Step 13: Test storage version migration

```bash
# When API version changes, storage version must migrate
# Check current storage version for a resource

kubectl get crd crontabs.stable.example.com -o jsonpath='{.status.storedVersions}'
echo ""

# Multiple stored versions indicate migration needed
```

## Step 14: Verify webhook compatibility

```bash
# Check if webhooks specify API versions
cat <<EOF | kubectl apply -f -
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: version-test-webhook
webhooks:
- name: version.test.com
  clientConfig:
    url: https://example.com/validate
  rules:
  - apiGroups: ["apps"]
    apiVersions: ["v1"]  # Explicit version!
    operations: ["CREATE"]
    resources: ["deployments"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
EOF
```

**Check webhook version support:**
```bash
kubectl get validatingwebhookconfigurations version-test-webhook -o jsonpath='{.webhooks[0].admissionReviewVersions}'
echo ""
```

## Step 15: Document upgrade path

```bash
cat > /tmp/upgrade-plan.md << 'EOF'
# Kubernetes Upgrade Plan

## Pre-Upgrade Checklist
- [ ] Backup etcd
- [ ] Check deprecated APIs
- [ ] Update CRDs to new versions
- [ ] Test upgrade in staging
- [ ] Review changelog for breaking changes
- [ ] Update client tools (kubectl, helm, etc.)

## Version Compatibility Rules
- Control plane can be 1 minor version ahead of nodes
- Nodes cannot be newer than control plane
- kubectl can be ±1 minor version from API server
- Upgrade one minor version at a time (1.26 → 1.27 → 1.28)

## Upgrade Order
1. Backup cluster state (etcd, configs)
2. Upgrade control plane components:
   - kube-apiserver
   - kube-controller-manager
   - kube-scheduler
3. Upgrade cluster add-ons (CNI, DNS, etc.)
4. Upgrade nodes (drain → upgrade → uncordon)
5. Verify cluster health
6. Update client tools

## Post-Upgrade Verification
- [ ] All nodes Ready
- [ ] All system pods Running
- [ ] API server responding
- [ ] Workloads functioning
- [ ] No deprecated API warnings
- [ ] Check logs for errors

## Rollback Plan
- Keep etcd backup for 24 hours
- Document current versions
- Test rollback procedure in staging
- Know the rollback limitations (some changes irreversible)
EOF

cat /tmp/upgrade-plan.md
```

## Key Observations

✅ **Version skew** - control plane can be N+1 ahead of nodes
✅ **kubectl compatibility** - ±1 version from API server
✅ **API deprecation** - old versions removed after grace period
✅ **CRD versions** - must support new API versions before upgrade
✅ **Storage migration** - required when API versions change
✅ **One minor version at a time** - skip versions = unsupported

## Production Patterns

**Safe upgrade process:**
```bash
# 1. Check current version
kubectl version --short

# 2. Review changelog
# https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/

# 3. Check for deprecated APIs
kubectl get --raw /openapi/v2 | jq '.definitions | keys[] | select(. | contains("beta"))' | head -20

# 4. Update CRDs first
kubectl apply -f updated-crds.yaml

# 5. Upgrade control plane (one component at a time)
# Managed clusters: Use cloud provider tools
# Self-managed: Upgrade binaries

# 6. Drain and upgrade nodes
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data
# Upgrade node kubelet
kubectl uncordon node-1

# 7. Verify
kubectl get nodes
kubectl get pods -A
kubectl get events --sort-by='.lastTimestamp' | head -20
```

**Automated deprecation check:**
```bash
# Use pluto to find deprecated APIs
# https://github.com/FairwindsOps/pluto

# Install pluto
curl -L "https://github.com/FairwindsOps/pluto/releases/download/v5.18.0/pluto_5.18.0_linux_amd64.tar.gz" | tar xz
./pluto detect-all-in-cluster --target-versions k8s=v1.29.0
```

**Version-aware YAML:**
```yaml
# Use current stable API versions
apiVersion: apps/v1  # Not apps/v1beta1
kind: Deployment

# Specify webhook versions
admissionReviewVersions: ["v1", "v1beta1"]  # Support both

# CRD with multiple versions
spec:
  versions:
  - name: v2
    served: true
    storage: true
  - name: v1
    served: true
    storage: false
    deprecated: true
```

**Monitoring for API deprecation:**
```yaml
# Prometheus alert
- alert: DeprecatedAPIUsage
  expr: apiserver_requested_deprecated_apis > 0
  for: 1h
  annotations:
    summary: "Deprecated API {{ $labels.group }}/{{ $labels.version }} in use"
```

## Cleanup

```bash
kubectl delete deployment api-version-test
kubectl delete crd crontabs.stable.example.com
kubectl delete crontab test-crontab 2>/dev/null
kubectl delete validatingwebhookconfiguration version-test-webhook
kubectl delete service legacy-service
rm -f /tmp/check-deprecated.sh /tmp/upgrade-precheck.sh /tmp/upgrade-plan.md
```

---
Next: Day 49 - Backup and Restore Failures with Velero
