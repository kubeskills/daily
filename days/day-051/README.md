## Day 51: API Server Overload - Rate Limit Hell

### Email Copy

**Subject:** Day 51: API Server Overload - When 429 Becomes Your Life

THE IDEA:
Flood the API server with requests and trigger aggressive rate limiting. Watch
controllers fail to reconcile, kubectl commands timeout, and cluster become unresponsive.

THE SETUP:
Generate massive API traffic, exceed flow control limits, and observe priority
queues rejecting requests. Test different user types and request priorities.

WHAT I LEARNED:
- API Priority and Fairness (APF) queues requests by priority
- System requests (controllers) prioritized over users
- Watch requests count toward rate limits
- List requests more expensive than Get
- Webhook calls can overwhelm API server

WHY IT MATTERS:
API server overload causes:
- Complete cluster control plane failure
- Unable to deploy or debug (all requests rejected)
- Controller loops failing (cluster state diverges)

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-051

Tomorrow: etcd corruption and quorum loss.

---
Chad

---

## Killercoda Lab Instructions



## Step 1: Check API server metrics

```bash
# Get API server metrics
kubectl get --raw /metrics | grep apiserver_request_total | head -20

# Check current inflight requests
kubectl get --raw /metrics | grep apiserver_current_inflight_requests
```

## Step 2: Check APF configuration

```bash
# List FlowSchemas (request routing)
kubectl get flowschema

# List PriorityLevelConfigurations (rate limits)
kubectl get prioritylevelconfiguration

# View default flow schema
kubectl get flowschema workload-high -o yaml
```

## Step 3: Generate moderate API load

```bash
# Make repeated requests
for i in $(seq 1 100); do
  kubectl get pods > /dev/null 2>&1 &
done
wait

echo "Burst complete"
```

## Step 4: Monitor API server request rates

```bash
# Check request metrics
kubectl get --raw /metrics | grep -E "apiserver_request_total|apiserver_flowcontrol"

# Check for throttling
kubectl get --raw /metrics | grep apiserver_flowcontrol_rejected_requests_total
```

## Step 5: Create custom PriorityLevelConfiguration

```bash
cat <<EOF | kubectl apply -f -
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: PriorityLevelConfiguration
metadata:
  name: test-priority
spec:
  type: Limited
  limited:
    nominalConcurrencyShares: 10
    limitResponse:
      type: Queue
      queuing:
        queues: 16
        queueLengthLimit: 50
        handSize: 4
EOF
```

## Step 6: Create custom FlowSchema

```bash
cat <<EOF | kubectl apply -f -
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: FlowSchema
metadata:
  name: test-flow
spec:
  distinguisherMethod:
    type: ByUser
  matchingPrecedence: 1000
  priorityLevelConfiguration:
    name: test-priority
  rules:
  - resourceRules:
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["list", "get"]
    subjects:
    - kind: User
      user:
        name: test-user
EOF
```

## Step 7: Test as different user (simulate)

```bash
# Create ServiceAccount
kubectl create serviceaccount load-test-sa

# Get token
TOKEN=$(kubectl create token load-test-sa --duration=1h)

# Make requests as this user
for i in $(seq 1 50); do
  kubectl --token=$TOKEN get pods > /dev/null 2>&1 &
done
wait
```

## Step 8: Generate heavy load (careful!)

```bash
cat > /tmp/api-load.sh << 'EOF'
#!/bin/bash
COUNT=0
while [ $COUNT -lt 500 ]; do
  kubectl get pods -A > /dev/null 2>&1 &
  kubectl get services -A > /dev/null 2>&1 &
  kubectl get deployments -A > /dev/null 2>&1 &

  COUNT=$((COUNT + 3))

  if [ $((COUNT % 50)) -eq 0 ]; then
    echo "Requests sent: $COUNT"
    sleep 0.5
  fi
done
wait
EOF

chmod +x /tmp/api-load.sh

# Run load test
/tmp/api-load.sh
```

**Check for 429 errors:**
```bash
kubectl get pods 2>&1 | grep -i "429\|too many"
```

## Step 9: Check APF queue status

```bash
# Get APF metrics
kubectl get --raw /metrics | grep apiserver_flowcontrol_request_queue_length

# Check rejections
kubectl get --raw /metrics | grep apiserver_flowcontrol_rejected_requests_total
```

## Step 10: Test watch requests

```bash
# Start multiple watches (expensive!)
for i in $(seq 1 10); do
  kubectl get pods -A --watch > /dev/null 2>&1 &
done

# Let them run for a bit
sleep 10

# Kill watches
pkill -f "kubectl.*watch"
```

**Check watch metrics:**
```bash
kubectl get --raw /metrics | grep apiserver_longrunning_requests
```

## Step 11: Test list vs get performance

```bash
# Time list operation (expensive)
time kubectl get pods -A > /dev/null

# Time get operation (cheap)
time kubectl get pod kube-apiserver-controlplane -n kube-system > /dev/null
```

List is much more expensive!

## Step 12: Simulate webhook overload

```bash
cat <<EOF | kubectl apply -f -
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: load-test-webhook
webhooks:
- name: load.test.com
  clientConfig:
    url: "https://nonexistent-webhook.example.com/validate"
  rules:
  - operations: ["CREATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  failurePolicy: Ignore
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 1
EOF

# Try to create pods (will timeout on webhook)
for i in $(seq 1 10); do
  kubectl run webhook-test-$i --image=nginx 2>&1 &
done

# Check for delays
kubectl get events --sort-by='.lastTimestamp' | grep webhook
```

## Step 13: Monitor API server CPU/memory

```bash
# Check API server pod
kubectl top pod -n kube-system -l component=kube-apiserver

# Check API server logs
kubectl logs -n kube-system -l component=kube-apiserver --tail=50 | grep -i "throttl\|limit\|reject"
```

## Step 14: Test request timeout

```bash
# Make request with short timeout
kubectl get pods --request-timeout=1s 2>&1 || echo "Timeout!"

# Make request with longer timeout
kubectl get pods --request-timeout=30s
```

## Step 15: Check priority levels

```bash
# View all priority levels
kubectl get prioritylevelconfiguration -o custom-columns=NAME:.metadata.name,SHARES:.spec.limited.nominalConcurrencyShares,QUEUES:.spec.limited.limitResponse.queuing.queues

# Check system vs user priorities
kubectl describe prioritylevelconfiguration system
kubectl describe prioritylevelconfiguration workload-high
```

## Key Observations

✅ **APF** - queues and prioritizes API requests
✅ **429 errors** - rate limit exceeded
✅ **System priority** - controllers have higher priority than users
✅ **Watch requests** - long-running, count toward limits
✅ **List expensive** - more costly than Get
✅ **Webhook timeouts** - can block API server

## Production Patterns

**Monitor API server health:**
```bash
# Check request latency
kubectl get --raw /metrics | grep apiserver_request_duration_seconds

# Check error rate
kubectl get --raw /metrics | grep 'apiserver_request_total.*code="5'

# Alert on high error rate
# rate(apiserver_request_total{code=~"5.."}[5m]) > 10
```

**Create priority for CI/CD:**
```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: PriorityLevelConfiguration
metadata:
  name: cicd-priority
spec:
  type: Limited
  limited:
    nominalConcurrencyShares: 30
    limitResponse:
      type: Queue
      queuing:
        queues: 64
        queueLengthLimit: 50
---
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: FlowSchema
metadata:
  name: cicd-flows
spec:
  matchingPrecedence: 500
  priorityLevelConfiguration:
    name: cicd-priority
  rules:
  - subjects:
    - kind: ServiceAccount
      serviceAccount:
        name: cicd-bot
        namespace: cicd
    resourceRules:
    - apiGroups: ["*"]
      resources: ["*"]
      verbs: ["*"]
```

**Implement client-side rate limiting:**
```bash
#!/bin/bash
# kubectl wrapper with rate limiting

MAX_REQUESTS=100
INTERVAL=60
COUNT_FILE="/tmp/kubectl_count"

# Initialize counter
if [ ! -f "$COUNT_FILE" ]; then
  echo "0 $(date +%s)" > $COUNT_FILE
fi

# Read counter
read count timestamp < $COUNT_FILE
current=$(date +%s)

# Reset if interval passed
if [ $((current - timestamp)) -gt $INTERVAL ]; then
  count=0
  timestamp=$current
fi

# Check limit
if [ $count -ge $MAX_REQUESTS ]; then
  echo "Rate limit exceeded. Wait $((INTERVAL - (current - timestamp))) seconds"
  exit 1
fi

# Increment and execute
echo "$((count + 1)) $timestamp" > $COUNT_FILE
exec kubectl "$@"
```

**Scale API server:**
```bash
# Check current replicas
kubectl get deployment -n kube-system kube-apiserver 2>/dev/null || echo "API server not managed as deployment"

# In managed clusters, increase API server instances
# GKE: --num-masters=3
# EKS: Multiple control plane endpoints
# AKS: Standard/Premium tier
```

**Webhook best practices:**
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: production-webhook
webhooks:
- name: validate.example.com
  failurePolicy: Ignore  # Don't block on failure
  timeoutSeconds: 3      # Fast timeout
  namespaceSelector:
    matchExpressions:
    - key: webhook-enabled
      operator: In
      values: ["true"]  # Only selected namespaces
  objectSelector:
    matchExpressions:
    - key: skip-webhook
      operator: DoesNotExist  # Allow opt-out
```

## Cleanup

```bash
kubectl delete flowschema test-flow
kubectl delete prioritylevelconfiguration test-priority
kubectl delete validatingwebhookconfiguration load-test-webhook
kubectl delete serviceaccount load-test-sa
kubectl delete pod webhook-test-{1..10} 2>/dev/null
rm -f /tmp/api-load.sh
```

---
Next: Day 52 - etcd Corruption and Quorum Loss
