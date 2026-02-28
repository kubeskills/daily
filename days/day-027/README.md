## Day 27: API Server Rate Limiting - Request Throttling

## THE IDEA:
Flood the API server with requests and watch it start rate limiting. Discover
how priority and fairness queues protect the cluster from runaway clients.

## THE SETUP:
Make rapid kubectl calls, trigger rate limiting, and observe 429 Too Many
Requests errors. Learn about APF (API Priority and Fairness) configuration.

## WHAT I LEARNED:

- API server has per-user rate limits
- Priority and Fairness (APF) queues requests by priority
- Watch requests don't count toward rate limit
- ServiceAccount can exhaust API quota

## WHY IT MATTERS:
Rate limiting issues cause:
- Controllers unable to reconcile due to quota exhaustion
- CI/CD pipelines failing with 429 errors
- Monitoring agents blocked from scraping metrics

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-027

Week 4 complete! 28 days of Kubernetes mastery unlocked!

---

# Killercoda Lab Instructions

## Step 1: Check API server rate limiting config

```bash
# Check if APF is enabled (default in K8s 1.20+)
kubectl get flowschemas
kubectl get prioritylevelconfigurations

```

## Step 2: Make baseline API requests

```bash
# Single request timing
time kubectl get pods

# Measure latency
for i in $(seq 1 10); do
  time kubectl get pods > /dev/null 2>&1
done

```

## Step 3: Flood API server with requests

```bash
# Rapid requests (be careful!)
for i in $(seq 1 100); do
  kubectl get pods > /dev/null 2>&1 &
done
wait

echo "Burst complete"

```

**Check for rate limit errors:**

```bash
kubectl get pods 2>&1 | grep -i "too many"

```

## Step 4: Trigger rate limiting deliberately

```bash
# Create many resources rapidly
for i in $(seq 1 500); do
  kubectl create configmap flood-$i --from-literal=key=value --dry-run=client -o yaml | kubectl apply -f - > /dev/null 2>&1 &
  if [ $((i % 50)) -eq 0 ]; then
    echo "Created $i configmaps"
    wait
  fi
done
wait

```

**Check for throttling:**

```bash
kubectl get events | grep -i throttl

```

## Step 5: Check API server metrics

```bash
# Get API server metrics (if accessible)
kubectl get --raw /metrics | grep apiserver_request | head -20

# Key metrics:
# apiserver_request_total
# apiserver_request_duration_seconds
# apiserver_current_inflight_requests

```

## Step 6: Examine FlowSchemas

```bash
kubectl get flowschemas

# View default flow schemas
kubectl get flowschema system-leader-election -o yaml
kubectl get flowschema workload-high -o yaml
kubectl get flowschema global-default -o yaml

```

**Key fields:**

- matchingPrecedence (lower = higher priority)
- priorityLevelConfiguration (request queue)
- distinguisherMethod (how to split users)

## Step 7: Check PriorityLevelConfigurations

```bash
kubectl get prioritylevelconfigurations

# View specific priority level
kubectl get prioritylevelconfiguration workload-high -o yaml

```

**Key fields:**

- nominalConcurrencyShares (relative weight)
- limitResponse.type (Queue or Reject)
- limitResponse.queuing (queue configuration)

## Step 8: Test with different user identities

```bash
# Create ServiceAccount
kubectl create serviceaccount api-user

# Create token
USER_TOKEN=$(kubectl create token api-user --duration=1h)

# Make requests as this user
kubectl --token=$USER_TOKEN get pods --request-timeout=5s

```

## Step 9: Create custom FlowSchema

```bash
cat <<EOF | kubectl apply -f -
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: FlowSchema
metadata:
  name: custom-priority
spec:
  distinguisherMethod:
    type: ByUser
  matchingPrecedence: 1000
  priorityLevelConfiguration:
    name: workload-low
  rules:
  - resourceRules:
    - apiGroups: [""]
      resources: ["configmaps"]
      verbs: ["list", "get"]
    subjects:
    - kind: ServiceAccount
      serviceAccount:
        name: api-user
        namespace: default
EOF

```

## Step 10: Test throttling with watch

```bash
# Watches typically exempt from rate limiting
kubectl get pods -w &
WATCH_PID=$!

sleep 5

# Kill watch
kill $WATCH_PID 2>/dev/null

```

## Step 11: Exhaust API quota deliberately

```bash
# Create script for sustained load
cat > /tmp/api-flood.sh << 'EOF'
#!/bin/bash
while true; do
  kubectl get pods > /dev/null 2>&1
  kubectl get services > /dev/null 2>&1
  kubectl get deployments > /dev/null 2>&1
done
EOF

chmod +x /tmp/api-flood.sh

# Run in background
/tmp/api-flood.sh &
FLOOD_PID=$!

# Let it run for 30 seconds
sleep 30

# Check for rate limiting
kubectl get pods 2>&1

# Stop flood
kill $FLOOD_PID 2>/dev/null

```

## Step 12: Check API server request queue

```bash
kubectl get --raw /metrics | grep apiserver_flowcontrol

```

Key metrics:

- `apiserver_flowcontrol_rejected_requests_total`
- `apiserver_flowcontrol_request_queue_length_after_enqueue`
- `apiserver_flowcontrol_current_executing_requests`

## Step 13: Test priority override

```bash
# System priority requests (controllers)
# These have higher priority than user requests

# Check system flow schemas
kubectl get flowschema system-nodes -o yaml
kubectl get flowschema kube-system-service-accounts -o yaml

```

## Step 14: Monitor inflight requests

```bash
kubectl get --raw /metrics | grep apiserver_current_inflight_requests

```

Shows:

- Current executing requests
- Max allowed (nominalConcurrencyShares)

## Step 15: Test with different request types

```bash
# List (typically limited)
time kubectl get pods

# Get (typically limited)
time kubectl get pod <pod-name>

# Watch (typically exempt)
kubectl get pods -w &
WATCH_PID=$!
sleep 5
kill $WATCH_PID 2>/dev/null

# Create (typically limited)
time kubectl create configmap test-rate --from-literal=key=val
kubectl delete configmap test-rate

```

## Key Observations

✅ **APF (API Priority and Fairness)** - queues and prioritizes requests
✅ **Per-user rate limiting** - each user/SA has separate quota
✅ **Watch requests exempt** - don't count toward rate limit
✅ **System requests prioritized** - controllers > users
✅ **429 errors** - Too Many Requests = rate limited

## Production Patterns

**Create priority for CI/CD:**

```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: PriorityLevelConfiguration
metadata:
  name: ci-cd-priority
spec:
  type: Limited
  limited:
    nominalConcurrencyShares: 50
    limitResponse:
      type: Queue
      queuing:
        queues: 64
        queueLengthLimit: 50
        handSize: 8
---
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: FlowSchema
metadata:
  name: ci-cd-flows
spec:
  matchingPrecedence: 500
  priorityLevelConfiguration:
    name: ci-cd-priority
  rules:
  - subjects:
    - kind: ServiceAccount
      serviceAccount:
        name: ci-cd-bot
        namespace: ci-cd
    resourceRules:
    - apiGroups: ["*"]
      resources: ["*"]
      verbs: ["*"]

```

**Monitor rate limiting:**

```bash
# Alert on high rejection rate
apiserver_flowcontrol_rejected_requests_total > 100

```

**Implement backoff in clients:**

```bash
#!/bin/bash
max_retries=5
retry=0

while [ $retry -lt $max_retries ]; do
  if kubectl get pods > /dev/null 2>&1; then
    break
  else
    retry=$((retry + 1))
    sleep $((2 ** retry))
  fi
done

```

## Cleanup

```bash
kubectl delete configmap $(kubectl get cm -o name | grep flood-) 2>/dev/null
kubectl delete flowschema custom-priority 2>/dev/null
kubectl delete serviceaccount api-user 2>/dev/null
rm -f /tmp/api-flood.sh

```

---

Next: Day 28 - Pod Security Admission