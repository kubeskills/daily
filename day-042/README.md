## Day 42: Logging Pipelines - Lost in Transit

## THE IDEA:
Deploy Fluentd/Fluent Bit to collect logs and watch them get lost due to buffer
overflow, parsing errors, or destination unavailability. Debug silent failures.

## THE SETUP:
Install log collector, generate high-volume logs, misconfigure parsers, and
break the backend. Discover why logs aren't reaching Elasticsearch/Loki.

## WHAT I LEARNED:
- Logs buffer in memory (data loss on crash)
- Parser errors drop entire log entries
- Back-pressure causes buffer overflow
- Container logs rotate (old logs lost)
- stdout != log file (need different collectors)

## WHY IT MATTERS:
Lost logs cause:
- Missing audit trail during incidents
- Compliance violations (SOC2, PCI-DSS)
- Inability to debug production issues

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-042

Tomorrow: Distributed tracing gaps and missing spans.


---

# Killercoda Lab Instructions


## Step 1: Check default container logging

```bash
# Deploy simple app
kubectl run logger --image=busybox --restart=Never -- sh -c 'while true; do echo "Log line $(date)"; sleep 1; done'

# Check logs
kubectl logs logger --tail=10
```

Logs available via `kubectl logs` (stored on node)

## Step 2: Check log location on node

```bash
# Find container ID
CONTAINER_ID=$(kubectl get pod logger -o jsonpath='{.status.containerStatuses[0].containerID}' | cut -d/ -f3)
echo "Container ID: $CONTAINER_ID"

# Logs are at (conceptually):
# /var/log/pods/<namespace>_<pod>_<uid>/<container>/*.log
```

## Step 3: Install Fluent Bit

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: logging
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluent-bit
  namespace: logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluent-bit
rules:
- apiGroups: [""]
  resources:
  - namespaces
  - pods
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluent-bit
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: fluent-bit
subjects:
- kind: ServiceAccount
  name: fluent-bit
  namespace: logging
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush        5
        Daemon       Off
        Log_Level    info

    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            docker
        Tag               kube.*
        Refresh_Interval  5
        Mem_Buf_Limit     5MB

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token

    [OUTPUT]
        Name   stdout
        Match  *

  parsers.conf: |
    [PARSER]
        Name   docker
        Format json
        Time_Key time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
        Time_Keep On
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:2.1
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc/
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: fluent-bit-config
        configMap:
          name: fluent-bit-config
EOF
```

**Wait for DaemonSet:**
```bash
kubectl wait --for=condition=Ready pods -l app=fluent-bit -n logging --timeout=120s
```

## Step 4: Check Fluent Bit logs

```bash
kubectl logs -n logging -l app=fluent-bit --tail=50
```

Should see logs being collected!

## Step 5: Test parser error (drops logs)

```bash
# Deploy app with non-JSON logs
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: unparseable-logs
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'while true; do echo "INVALID-JSON-LOG $(date)"; sleep 1; done']
EOF
```

**Check Fluent Bit logs:**
```bash
kubectl logs -n logging -l app=fluent-bit --tail=100 | grep -i error
```

Parser errors for unparseable logs!

## Step 6: Fix with regex parser

```bash
kubectl patch configmap fluent-bit-config -n logging --type=merge -p '
{
  "data": {
    "parsers.conf": "[PARSER]\n    Name   docker\n    Format json\n    Time_Key time\n    Time_Format %Y-%m-%dT%H:%M:%S.%L\n\n[PARSER]\n    Name   plain\n    Format regex\n    Regex  ^(?<log>.*)$\n"
  }
}'

# Restart Fluent Bit
kubectl rollout restart daemonset fluent-bit -n logging
```

## Step 7: Test buffer overflow

```bash
# Generate massive logs
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: log-flood
spec:
  containers:
  - name: flood
    image: busybox
    command: ['sh', '-c']
    args:
    - |
      while true; do
        for i in $(seq 1 1000); do
          echo "Log flood message $i at $(date)"
        done
        sleep 0.1
      done
EOF
```

**Check Fluent Bit memory:**
```bash
kubectl top pod -n logging
```

**Check for buffer errors:**
```bash
kubectl logs -n logging -l app=fluent-bit --tail=100 | grep -i "buffer\|overflow\|backpressure"
```

## Step 8: Test log destination failure

```bash
# Update config to send to nonexistent Elasticsearch
kubectl patch configmap fluent-bit-config -n logging --type=merge -p '
{
  "data": {
    "fluent-bit.conf": "[SERVICE]\n    Flush 5\n\n[INPUT]\n    Name tail\n    Path /var/log/containers/*.log\n    Parser docker\n    Tag kube.*\n\n[OUTPUT]\n    Name es\n    Match *\n    Host nonexistent-elasticsearch\n    Port 9200\n    Retry_Limit 5\n"
  }
}'

# Restart
kubectl rollout restart daemonset fluent-bit -n logging
kubectl wait --for=condition=Ready pods -l app=fluent-bit -n logging --timeout=120s
```

**Check logs:**
```bash
kubectl logs -n logging -l app=fluent-bit --tail=50 | grep -i "connection\|failed\|error"
```

Connection failures! Logs being dropped.

## Step 9: Test log rotation

```bash
# Check container log rotation settings
docker info 2>/dev/null | grep -A 10 "Logging Driver" || echo "Docker not accessible"

# Conceptual: default is 10MB per file, 3 files
# Logs rotate = old logs lost
```

## Step 10: Test multiline log parsing

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multiline-logs
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c']
    args:
    - |
      while true; do
        echo "Exception in thread main:"
        echo "  at line 1"
        echo "  at line 2"
        echo "  at line 3"
        sleep 5
      done
EOF
```

Without multiline parser, each line is separate log entry!

## Step 11: Configure multiline parser

```bash
kubectl patch configmap fluent-bit-config -n logging --type=merge -p '
{
  "data": {
    "parsers.conf": "[PARSER]\n    Name   docker\n    Format json\n\n[PARSER]\n    Name   multiline\n    Format regex\n    Regex  /^(?<log>Exception.*)$/\n"
  }
}'
```

## Step 12: Test log filtering

```bash
# Add filter to drop debug logs
kubectl patch configmap fluent-bit-config -n logging --type=merge -p '
{
  "data": {
    "fluent-bit.conf": "[SERVICE]\n    Flush 5\n\n[INPUT]\n    Name tail\n    Path /var/log/containers/*.log\n    Parser docker\n    Tag kube.*\n\n[FILTER]\n    Name grep\n    Match kube.*\n    Exclude log DEBUG\n\n[OUTPUT]\n    Name stdout\n    Match *\n"
  }
}'

kubectl rollout restart daemonset fluent-bit -n logging
```

## Step 13: Test log enrichment

```bash
# Add Kubernetes metadata
kubectl patch configmap fluent-bit-config -n logging --type=merge -p '
{
  "data": {
    "fluent-bit.conf": "[SERVICE]\n    Flush 5\n\n[INPUT]\n    Name tail\n    Path /var/log/containers/*.log\n    Parser docker\n    Tag kube.*\n\n[FILTER]\n    Name kubernetes\n    Match kube.*\n    Kube_URL https://kubernetes.default.svc:443\n    Merge_Log On\n    K8S-Logging.Parser On\n    K8S-Logging.Exclude On\n\n[OUTPUT]\n    Name stdout\n    Match *\n    Format json_lines\n"
  }
}'

kubectl rollout restart daemonset fluent-bit -n logging
```

**Check enriched logs:**
```bash
kubectl logs -n logging -l app=fluent-bit --tail=20
```

Contains pod name, namespace, labels!

## Step 14: Test sidecar logging pattern

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-logging
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'while true; do echo "App log" >> /var/log/app.log; sleep 1; done']
    volumeMounts:
    - name: logs
      mountPath: /var/log
  - name: log-shipper
    image: busybox
    command: ['sh', '-c', 'tail -f /var/log/app.log']
    volumeMounts:
    - name: logs
      mountPath: /var/log
  volumes:
  - name: logs
    emptyDir: {}
EOF
```

**Check sidecar logs:**
```bash
kubectl logs sidecar-logging -c log-shipper
```

## Step 15: Debug missing logs

```bash
# Check if logs exist in container
kubectl exec logger -- sh -c 'echo "Direct log line"'
kubectl logs logger --tail=5

# Check Fluent Bit is reading logs
kubectl logs -n logging -l app=fluent-bit --tail=100 | grep logger

# Check for errors
kubectl logs -n logging -l app=fluent-bit --tail=200 | grep -i error

# Check buffer status
kubectl exec -n logging -l app=fluent-bit -- ls -lh /fluent-bit/tail/ 2>/dev/null || echo "No tail buffer"
```

## Key Observations

✅ **Parser errors** - drop entire log entries silently
✅ **Buffer limits** - cause data loss under high volume
✅ **Destination failure** - logs lost if backend unavailable
✅ **Log rotation** - old logs deleted from node
✅ **Multiline logs** - split into separate entries without parser
✅ **stdout vs files** - different collection strategies

## Production Patterns

**Robust Fluent Bit config:**
```yaml
[SERVICE]
    Flush         5
    Daemon        Off
    Log_Level     warn
    storage.path  /var/fluent-bit/state

[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    Parser            docker
    Tag               kube.*
    Mem_Buf_Limit     10MB
    Skip_Long_Lines   On
    Refresh_Interval  10
    storage.type      filesystem

[FILTER]
    Name                kubernetes
    Match               kube.*
    Merge_Log           On
    Keep_Log            Off
    K8S-Logging.Parser  On
    K8S-Logging.Exclude On

[OUTPUT]
    Name                 es
    Match                *
    Host                 elasticsearch
    Port                 9200
    Retry_Limit          5
    storage.total_limit_size 5G
```

**High-volume logging:**
```yaml
[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    Mem_Buf_Limit     50MB
    storage.type      filesystem
    storage.max_chunks_up 128

[OUTPUT]
    Name          forward
    Match         *
    Host          log-aggregator
    Port          24224
    Retry_Limit   False
```

**Log sampling (reduce volume):**
```yaml
[FILTER]
    Name          sampling
    Match         kube.*
    Percentage    10  # Only keep 10% of logs
```

## Cleanup

```bash
kubectl delete namespace logging
kubectl delete pod logger unparseable-logs log-flood multiline-logs sidecar-logging
```

---
