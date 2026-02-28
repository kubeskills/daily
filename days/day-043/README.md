## Day 43: Distributed Tracing - Missing Spans

## THE IDEA:
Deploy Jaeger for distributed tracing and watch traces become incomplete when
services don't propagate headers, sampling drops spans, or collector is down.

## THE SETUP:
Install Jaeger, create microservices with OpenTelemetry, break header propagation,
and misconfigure sampling. Debug why distributed traces have gaps.

## WHAT I LEARNED:
- Trace context must be propagated via HTTP headers
- Sampling drops spans (probabilistic or rate-limited)
- Missing instrumentation creates trace gaps
- Jaeger collector can be overwhelmed (back-pressure)
- baggage vs trace context (different purposes)

## WHY IT MATTERS:
Missing trace spans cause:
- Incomplete view of request flow
- Hidden performance bottlenecks
- Inability to debug cross-service issues

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-043

Tomorrow: Debugging slow pods with kubectl top and resource metrics.


---

# Killercoda Lab Instructions


## Step 1: Install Jaeger (all-in-one)

```bash
kubectl create namespace tracing

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: tracing
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
      - name: jaeger
        image: jaegertracing/all-in-one:1.50
        env:
        - name: COLLECTOR_OTLP_ENABLED
          value: "true"
        ports:
        - containerPort: 16686  # UI
        - containerPort: 4317   # OTLP gRPC
        - containerPort: 4318   # OTLP HTTP
        - containerPort: 14268  # Jaeger HTTP
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger
  namespace: tracing
spec:
  selector:
    app: jaeger
  ports:
  - name: ui
    port: 16686
    targetPort: 16686
  - name: otlp-grpc
    port: 4317
    targetPort: 4317
  - name: otlp-http
    port: 4318
    targetPort: 4318
  - name: jaeger-http
    port: 14268
    targetPort: 14268
EOF
```

**Expose Jaeger UI:**
```bash
kubectl port-forward -n tracing svc/jaeger 16686:16686 > /dev/null 2>&1 &
# Access at http://localhost:16686
```

## Step 2: Deploy service A (with tracing)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-a
spec:
  replicas: 1
  selector:
    matchLabels:
      app: service-a
  template:
    metadata:
      labels:
        app: service-a
    spec:
      containers:
      - name: app
        image: curlimages/curl:latest
        command: ['sh', '-c']
        args:
        - |
          while true; do
            # Simulate traced request to service-b
            TRACE_ID=\$(cat /proc/sys/kernel/random/uuid | cut -d- -f1)
            SPAN_ID=\$(cat /proc/sys/kernel/random/uuid | cut -d- -f1 | cut -c1-16)

            echo "Starting trace: \$TRACE_ID"
            curl -H "traceparent: 00-\$TRACE_ID-\$SPAN_ID-01" \
                 http://service-b:8080/ 2>/dev/null || echo "Service B unreachable"

            sleep 10
          done
---
apiVersion: v1
kind: Service
metadata:
  name: service-a
spec:
  selector:
    app: service-a
  ports:
  - port: 8080
EOF
```

## Step 3: Deploy service B WITHOUT header propagation (broken!)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-b-broken
spec:
  replicas: 1
  selector:
    matchLabels:
      app: service-b
  template:
    metadata:
      labels:
        app: service-b
        version: broken
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args:
        - "-text=Service B - NO HEADER PROPAGATION"
        - "-listen=:8080"
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: service-b
spec:
  selector:
    app: service-b
  ports:
  - port: 8080
    targetPort: 8080
EOF
```

**Check Jaeger UI:**
```bash
# Traces stop at service-a, no continuation to service-b
```

## Step 4: Deploy service B WITH header propagation (fixed)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-b-fixed
spec:
  replicas: 1
  selector:
    matchLabels:
      app: service-b
      version: fixed
  template:
    metadata:
      labels:
        app: service-b
        version: fixed
    spec:
      containers:
      - name: app
        image: curlimages/curl:latest
        command: ['sh', '-c']
        args:
        - |
          # Simple HTTP server that propagates traceparent
          while true; do
            nc -l -p 8080 -e sh -c '
              echo "HTTP/1.1 200 OK"
              echo "Content-Type: text/plain"
              echo ""
              echo "Service B - Propagated trace headers"
            '
          done
        ports:
        - containerPort: 8080
EOF
```

**Update service selector:**
```bash
kubectl patch service service-b --type=merge -p '{"spec":{"selector":{"app":"service-b","version":"fixed"}}}'
```

## Step 5: Test sampling rate (drops spans)

```bash
# Conceptual - typically set via environment variables:
# OTEL_TRACES_SAMPLER=parentbased_traceidratio
# OTEL_TRACES_SAMPLER_ARG=0.1  # 10% sampling
```

## Step 6: Deploy instrumented app with OpenTelemetry

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-config
data:
  otel-config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
    processors:
      batch:
        timeout: 1s
        send_batch_size: 1024
    exporters:
      otlp:
        endpoint: jaeger.tracing.svc:4317
        tls:
          insecure: true
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [otlp]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
spec:
  replicas: 1
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
      - name: collector
        image: otel/opentelemetry-collector:0.88.0
        args: ["--config=/etc/otel/config.yaml"]
        ports:
        - containerPort: 4317
        volumeMounts:
        - name: config
          mountPath: /etc/otel
      volumes:
      - name: config
        configMap:
          name: otel-config
---
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
spec:
  selector:
    app: otel-collector
  ports:
  - port: 4317
    targetPort: 4317
EOF
```

## Step 7: Test missing instrumentation (gaps in trace)

```bash
# Service C with NO instrumentation
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-c-uninstrumented
spec:
  replicas: 1
  selector:
    matchLabels:
      app: service-c
  template:
    metadata:
      labels:
        app: service-c
    spec:
      containers:
      - name: app
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: service-c
spec:
  selector:
    app: service-c
  ports:
  - port: 80
EOF
```

Trace shows call TO service-c but not INSIDE it!

## Step 8: Test collector back-pressure

```bash
# Generate massive number of spans
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: span-flood
spec:
  containers:
  - name: flood
    image: curlimages/curl:latest
    command: ['sh', '-c']
    args:
    - |
      while true; do
        for i in \$(seq 1 100); do
          TRACE_ID=\$(cat /proc/sys/kernel/random/uuid | cut -d- -f1)
          curl -X POST http://otel-collector:4317/v1/traces \
            -H "Content-Type: application/json" \
            -d "{\"traceId\":\"\$TRACE_ID\"}" 2>/dev/null &
        done
        sleep 0.1
      done
EOF
```

**Check collector logs:**
```bash
kubectl logs -l app=otel-collector --tail=50 | grep -i "buffer\|drop\|backpressure"
```

## Step 9: Test trace context formats

```bash
# W3C Trace Context (standard)
# traceparent: 00-<trace-id>-<span-id>-<flags>

# Jaeger format (legacy)
# uber-trace-id: <trace-id>:<span-id>:<parent-span-id>:<flags>

# B3 format (Zipkin)
# X-B3-TraceId: <trace-id>
# X-B3-SpanId: <span-id>
```

## Step 10: Test baggage propagation

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: baggage-test
spec:
  containers:
  - name: test
    image: curlimages/curl:latest
    command: ['sh', '-c']
    args:
    - |
      # Send request with baggage
      curl -H "baggage: user-id=12345,session-id=abc" \
           -H "traceparent: 00-trace123-span123-01" \
           http://service-b:8080/
      sleep 3600
EOF
```

Baggage carries user context through trace!

## Step 11: Check trace sampling strategies

```bash
# Probabilistic: sample X% of traces
# OTEL_TRACES_SAMPLER=traceidratio
# OTEL_TRACES_SAMPLER_ARG=0.01  # 1%

# Rate limiting: sample X traces per second
# OTEL_TRACES_SAMPLER=parentbased_traceidratio

# Always sample errors
# Custom sampler based on status code
```

## Step 12: Debug missing spans

```bash
# Check if service is sending spans
kubectl logs -l app=service-a --tail=50 | grep -i trace

# Check if collector is receiving
kubectl logs -l app=otel-collector --tail=100

# Check Jaeger storage
kubectl logs -n tracing -l app=jaeger --tail=100

# Query Jaeger API
curl -s "http://localhost:16686/api/traces?service=service-a&limit=10" | jq .
```

## Step 13: Test span attributes

```bash
# Spans should include:
# - service.name
# - http.method
# - http.status_code
# - http.url
# - db.statement (for DB queries)
# - error (boolean)
```

## Step 14: Test trace visualization

```bash
# Access Jaeger UI at http://localhost:16686
# - Service dependency graph
# - Trace timeline
# - Span details
# - Compare traces
```

## Step 15: Test long-running traces

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: slow-service
spec:
  containers:
  - name: app
    image: curlimages/curl:latest
    command: ['sh', '-c']
    args:
    - |
      while true; do
        nc -l -p 8080 -e sh -c '
          sleep 30  # Simulate slow operation
          echo "HTTP/1.1 200 OK"
          echo ""
          echo "Slow response"
        '
      done
EOF
```

Trace shows 30s span duration!

## Key Observations

✅ **Header propagation** - must pass traceparent/uber-trace-id
✅ **Sampling** - drops percentage of traces
✅ **Missing instrumentation** - creates trace gaps
✅ **Collector limits** - back-pressure drops spans
✅ **Context formats** - W3C vs Jaeger vs B3
✅ **Baggage** - carries user context (different from trace context)

## Production Patterns

**Complete tracing setup:**
```yaml
# Application with OpenTelemetry SDK
env:
- name: OTEL_SERVICE_NAME
  value: "my-service"
- name: OTEL_EXPORTER_OTLP_ENDPOINT
  value: "http://otel-collector:4317"
- name: OTEL_TRACES_SAMPLER
  value: "parentbased_traceidratio"
- name: OTEL_TRACES_SAMPLER_ARG
  value: "0.1"  # 10% sampling
- name: OTEL_RESOURCE_ATTRIBUTES
  value: "deployment.environment=production"
```

**OpenTelemetry Collector config:**
```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
  tail_sampling:
    policies:
      - name: error-traces
        type: status_code
        status_code: {status_codes: [ERROR]}
      - name: slow-traces
        type: latency
        latency: {threshold_ms: 5000}

exporters:
  otlp:
    endpoint: jaeger:4317
  prometheus:
    endpoint: 0.0.0.0:8889

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, tail_sampling]
      exporters: [otlp, prometheus]
```

## Cleanup

```bash
pkill -f "port-forward.*jaeger"
kubectl delete namespace tracing
kubectl delete deployment service-a service-b-broken service-b-fixed service-c-uninstrumented otel-collector
kubectl delete service service-a service-b service-c otel-collector
kubectl delete pod span-flood baggage-test slow-service 2>/dev/null
kubectl delete configmap otel-config
```

---
