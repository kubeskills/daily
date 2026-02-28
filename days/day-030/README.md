## Day 30: Operator Reconciliation - Infinite Loops

## THE IDEA:
Build a simple operator that creates a ConfigMap for every custom resource. 
Introduce a bug that causes infinite reconciliation loops and watch resource 
versions increment endlessly.

## THE SETUP:
Deploy an operator using kubebuilder patterns, trigger reconciliation, and 
observe what happens when you forget to update status or create duplicate 
resources.

## WHAT I LEARNED:
- Operators watch for changes and reconcile to desired state
- Infinite loops happen when reconciliation changes trigger more reconciliation
- Status subresource prevents reconciliation on status updates
- Owner references enable garbage collection

## WHY IT MATTERS:
Operator bugs cause:
- API server overload from rapid reconciliation
- etcd performance degradation from constant writes
- Resource thrashing (create/delete cycles)

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-030

Tomorrow: Helm chart templating errors blocking installations.

---


# Killercoda Lab Instructions


## Step 1: Create CRD for operator testing

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: webapps.apps.example.com
spec:
  group: apps.example.com
  names:
    kind: WebApp
    plural: webapps
    singular: webapp
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    subresources:
      status: {}
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: [message]
            properties:
              message:
                type: string
              replicas:
                type: integer
                default: 1
          status:
            type: object
            properties:
              configMapName:
                type: string
              phase:
                type: string
              observedGeneration:
                type: integer
EOF
```

## Step 2: Create simple reconciliation script (Python)

```bash
cat > /tmp/operator.py << 'EOF'
#!/usr/bin/env python3
import time
import json
from kubernetes import client, config, watch

config.load_incluster_config() if config.load_incluster_config else config.load_kube_config()
v1 = client.CoreV1Api()
custom_api = client.CustomObjectsApi()

GROUP = "apps.example.com"
VERSION = "v1"
PLURAL = "webapps"

def reconcile_forever():
    w = watch.Watch()
    print("Starting operator watch loop...")
    
    for event in w.stream(custom_api.list_cluster_custom_object, GROUP, VERSION, PLURAL):
        obj = event['object']
        event_type = event['type']
        name = obj['metadata']['name']
        namespace = obj['metadata'].get('namespace', 'default')
        spec = obj.get('spec', {})
        
        print(f"Event: {event_type} - {namespace}/{name}")
        
        if event_type in ['ADDED', 'MODIFIED']:
            reconcile(namespace, name, spec, obj)

def reconcile(namespace, name, spec, obj):
    message = spec.get('message', 'default')
    cm_name = f"{name}-config"
    
    # BUG: Creating ConfigMap without checking if it exists
    # This causes infinite reconciliation!
    
    try:
        configmap = client.V1ConfigMap(
            metadata=client.V1ObjectMeta(
                name=cm_name,
                namespace=namespace,
                owner_references=[{
                    'apiVersion': f'{GROUP}/{VERSION}',
                    'kind': 'WebApp',
                    'name': name,
                    'uid': obj['metadata']['uid'],
                    'controller': True
                }]
            ),
            data={'message': message}
        )
        
        v1.create_namespaced_config_map(namespace, configmap)
        print(f"Created ConfigMap: {cm_name}")
        
        # Update status (causes another MODIFIED event!)
        status_body = {
            'status': {
                'configMapName': cm_name,
                'phase': 'Ready'
            }
        }
        custom_api.patch_namespaced_custom_object_status(
            GROUP, VERSION, namespace, PLURAL, name, status_body
        )
        
    except client.exceptions.ApiException as e:
        if e.status == 409:
            print(f"ConfigMap {cm_name} already exists")
        else:
            print(f"Error: {e}")

if __name__ == '__main__':
    reconcile_forever()
EOF

chmod +x /tmp/operator.py
```

## Step 3: Create operator deployment (simplified)

```bash
# Note: This is a conceptual example
# Real operators would run in-cluster with proper RBAC

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: webapp-operator
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: webapp-operator
rules:
- apiGroups: ["apps.example.com"]
  resources: ["webapps", "webapps/status"]
  verbs: ["get", "list", "watch", "update", "patch"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "create", "update", "delete"]
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
  namespace: default
EOF
```

## Step 4: Simulate operator behavior manually

```bash
# Create WebApp
cat <<EOF | kubectl apply -f -
apiVersion: apps.example.com/v1
kind: WebApp
metadata:
  name: test-app
spec:
  message: "Hello World"
  replicas: 1
EOF
```

## Step 5: Manually reconcile (creating ConfigMap)

```bash
# First reconciliation - create ConfigMap
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-app-config
  ownerReferences:
  - apiVersion: apps.example.com/v1
    kind: WebApp
    name: test-app
    uid: $(kubectl get webapp test-app -o jsonpath='{.metadata.uid}')
    controller: true
data:
  message: "Hello World"
EOF
```

## Step 6: Watch for reconciliation trigger

```bash
# Get initial resource version
kubectl get webapp test-app -o jsonpath='{.metadata.resourceVersion}'
echo ""

# Update status (without subresource - wrong!)
kubectl patch webapp test-app --type=merge -p '{"status":{"phase":"Ready"}}'

# Check resource version again
kubectl get webapp test-app -o jsonpath='{.metadata.resourceVersion}'
echo ""
```

**Resource version changed!** This would trigger reconciliation.

## Step 7: Correct way - use status subresource

```bash
# This does NOT change spec resourceVersion
kubectl patch webapp test-app --subresource=status --type=merge -p '{"status":{"phase":"Ready","configMapName":"test-app-config"}}'

# Check resource version
kubectl get webapp test-app -o jsonpath='{.metadata.resourceVersion}'
echo ""
```

Spec resourceVersion unchanged!

## Step 8: Test observedGeneration pattern

```bash
# Get current generation
GENERATION=$(kubectl get webapp test-app -o jsonpath='{.metadata.generation}')
echo "Generation: $GENERATION"

# Update status with observedGeneration
kubectl patch webapp test-app --subresource=status --type=merge -p "{\"status\":{\"observedGeneration\":$GENERATION}}"

# Operator should skip if observedGeneration == generation
```

## Step 9: Simulate infinite loop scenario

```bash
# Create script that mimics bad operator behavior
cat > /tmp/bad-operator.sh << 'EOF'
#!/bin/bash
count=0
while [ $count -lt 20 ]; do
  # Get webapp
  MESSAGE=$(kubectl get webapp test-app -o jsonpath='{.spec.message}')
  
  # Create/update ConfigMap (triggers change)
  kubectl create configmap test-app-config \
    --from-literal=message="$MESSAGE" \
    --dry-run=client -o yaml | kubectl apply -f -
  
  # Update spec (WRONG - triggers reconciliation)
  kubectl patch webapp test-app --type=merge -p "{\"spec\":{\"lastReconciled\":\"$(date +%s)\"}}" 2>/dev/null
  
  count=$((count + 1))
  echo "Reconciliation $count"
  sleep 1
done
EOF

chmod +x /tmp/bad-operator.sh
```

**Run it:**
```bash
/tmp/bad-operator.sh
```

Watch resource version increment rapidly!

## Step 10: Check event storm

```bash
kubectl get events --sort-by='.lastTimestamp' | grep test-app | tail -20
```

## Step 11: Test owner reference garbage collection

```bash
# Delete WebApp
kubectl delete webapp test-app

# ConfigMap should be deleted automatically
sleep 2
kubectl get configmap test-app-config 2>&1 || echo "ConfigMap deleted (garbage collected)"
```

## Step 12: Test finalizer blocking deletion

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps.example.com/v1
kind: WebApp
metadata:
  name: with-finalizer
  finalizers:
  - apps.example.com/cleanup
spec:
  message: "Testing finalizers"
EOF
```

**Try to delete:**
```bash
kubectl delete webapp with-finalizer
```

**Stuck in Terminating!**

```bash
kubectl get webapp with-finalizer
```

**Remove finalizer:**
```bash
kubectl patch webapp with-finalizer --type=json -p='[{"op": "remove", "path": "/metadata/finalizers"}]'
```

Now it deletes.

## Step 13: Test reconciliation with conditions

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps.example.com/v1
kind: WebApp
metadata:
  name: with-conditions
spec:
  message: "Testing conditions"
EOF

# Update status with conditions
kubectl patch webapp with-conditions --subresource=status --type=merge -p '{
  "status": {
    "phase": "Creating",
    "conditions": [
      {
        "type": "ConfigMapReady",
        "status": "False",
        "reason": "Creating",
        "message": "ConfigMap is being created",
        "lastTransitionTime": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"
      }
    ]
  }
}'

kubectl get webapp with-conditions -o yaml | grep -A 10 status
```

## Step 14: Simulate controller with retry backoff

```bash
cat > /tmp/smart-reconcile.sh << 'EOF'
#!/bin/bash
WEBAPP=$1
RETRY_COUNT=0
MAX_RETRIES=3

reconcile() {
  echo "Reconciling $WEBAPP (attempt $RETRY_COUNT)"
  
  # Check if ConfigMap exists
  if kubectl get configmap ${WEBAPP}-config &>/dev/null; then
    echo "ConfigMap exists, checking if update needed"
    # Only update if message changed
    return 0
  else
    echo "Creating ConfigMap"
    MESSAGE=$(kubectl get webapp $WEBAPP -o jsonpath='{.spec.message}')
    kubectl create configmap ${WEBAPP}-config --from-literal=message="$MESSAGE"
  fi
}

while [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
  if reconcile; then
    echo "Reconciliation successful"
    exit 0
  else
    RETRY_COUNT=$((RETRY_COUNT + 1))
    BACKOFF=$((2 ** RETRY_COUNT))
    echo "Reconciliation failed, retrying in ${BACKOFF}s"
    sleep $BACKOFF
  fi
done

echo "Reconciliation failed after $MAX_RETRIES attempts"
exit 1
EOF

chmod +x /tmp/smart-reconcile.sh
/tmp/smart-reconcile.sh with-conditions
```

## Step 15: Monitor reconciliation performance

```bash
# Check how many times object was updated
kubectl get webapp with-conditions -o jsonpath='{.metadata.resourceVersion}'
echo ""

# Check generation
kubectl get webapp with-conditions -o jsonpath='{.metadata.generation}'
echo ""

# Update spec (increments generation)
kubectl patch webapp with-conditions --type=merge -p '{"spec":{"message":"Updated"}}'

# Check generation again
kubectl get webapp with-conditions -o jsonpath='{.metadata.generation}'
echo ""
```

## Key Observations

✅ **Status subresource** - prevents reconciliation on status updates
✅ **observedGeneration** - tracks which generation was reconciled
✅ **Owner references** - enable garbage collection
✅ **Finalizers** - block deletion until cleanup completes
✅ **Conditions** - structured status information
✅ **Infinite loops** - happen when reconciliation triggers itself

## Production Patterns

**Proper reconciliation logic:**
```python
def reconcile(namespace, name, obj):
    spec = obj['spec']
    status = obj.get('status', {})
    
    # Check if already reconciled
    current_gen = obj['metadata']['generation']
    observed_gen = status.get('observedGeneration', 0)
    
    if current_gen == observed_gen:
        print(f"Already reconciled generation {current_gen}")
        return
    
    # Do actual work
    try:
        ensure_configmap_exists(namespace, name, spec)
        
        # Update status only (using subresource)
        update_status(namespace, name, {
            'phase': 'Ready',
            'observedGeneration': current_gen
        })
    except Exception as e:
        update_status(namespace, name, {
            'phase': 'Failed',
            'message': str(e)
        })
```

**Idempotent resource creation:**
```python
def ensure_configmap_exists(namespace, name, spec):
    cm_name = f"{name}-config"
    
    try:
        existing = v1.read_namespaced_config_map(cm_name, namespace)
        # Check if update needed
        if existing.data.get('message') != spec['message']:
            existing.data['message'] = spec['message']
            v1.replace_namespaced_config_map(cm_name, namespace, existing)
    except ApiException as e:
        if e.status == 404:
            # Doesn't exist, create it
            v1.create_namespaced_config_map(namespace, configmap_body)
        else:
            raise
```

## Cleanup

```bash
kubectl delete webapp --all
kubectl delete configmap test-app-config with-conditions-config 2>/dev/null
kubectl delete crd webapps.apps.example.com
kubectl delete clusterrolebinding webapp-operator 2>/dev/null
kubectl delete clusterrole webapp-operator 2>/dev/null
kubectl delete serviceaccount webapp-operator 2>/dev/null
rm -f /tmp/operator.py /tmp/bad-operator.sh /tmp/smart-reconcile.sh
```

---
Next: Day 31 - Helm Chart Templating Errors