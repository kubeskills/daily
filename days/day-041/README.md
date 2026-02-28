## Day 41: Prometheus Metrics - Missing Targets

## THE IDEA:
Deploy Prometheus to scrape pod metrics and watch targets go missing from
ServiceMonitor misconfiguration, missing ports, or NetworkPolicy blocking.

## THE SETUP:
Install Prometheus Operator, create ServiceMonitors with wrong labels, missing
ports, and blocked network access. Debug why metrics aren't being collected.

## WHAT I LEARNED:
- ServiceMonitor selects services by labels
- Service must have named port matching ServiceMonitor
- NetworkPolicy can block Prometheus scraping
- Prometheus uses service discovery to find targets
- Relabeling can break target discovery

## WHY IT MATTERS:
Missing metrics cause:
- Blind spots in monitoring (no alerts fire)
- Incomplete dashboards showing partial data
- SLA violations from undetected issues

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-041

Tomorrow: Logging pipeline failures and lost logs.


---

# Killercoda Lab Instructions


## Step 1: Install Prometheus Operator

```bash
kubectl create namespace monitoring

kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/v0.68.0/bundle.yaml

# Wait for operator
kubectl wait --for=condition=Ready pods -l app.kubernetes.io/name=prometheus-operator -n default --timeout=120s
```

## Step 2: Create Prometheus instance

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
---
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchLabels:
      team: frontend
  resources:
    requests:
      memory: 400Mi
  enableAdminAPI: false
EOF
```

## Step 3: Expose Prometheus UI

```bash
kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090 > /dev/null 2>&1 &
```

## Step 4: Deploy app with metrics endpoint

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: metrics-app
  template:
    metadata:
      labels:
        app: metrics-app
    spec:
      containers:
      - name: app
        image: nginx
        ports:
        - name: web
          containerPort: 80
        - name: metrics
          containerPort: 9113
---
apiVersion: v1
kind: Service
metadata:
  name: metrics-app-svc
  labels:
    app: metrics-app
spec:
  selector:
    app: metrics-app
  ports:
  - name: web
    port: 80
    targetPort: web
  - name: metrics
    port: 9113
    targetPort: metrics
EOF
```

## Step 5: Create ServiceMonitor with wrong label (fails!)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: wrong-label-monitor
  namespace: default
  labels:
    team: backend  # Wrong! Prometheus wants team: frontend
spec:
  selector:
    matchLabels:
      app: metrics-app
  endpoints:
  - port: metrics
    interval: 30s
EOF
```

**Check Prometheus targets:**
```bash
# Access http://localhost:9090/targets
# No targets appear!
```

## Step 6: Fix ServiceMonitor label

```bash
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: correct-label-monitor
  namespace: default
  labels:
    team: frontend  # Matches Prometheus serviceMonitorSelector
spec:
  selector:
    matchLabels:
      app: metrics-app
  endpoints:
  - port: metrics
    interval: 30s
EOF
```

**Check targets again:**
```bash
# Targets should appear now (though metrics endpoint doesn't exist yet)
```

## Step 7: Test missing port name

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: no-port-name-svc
  labels:
    app: no-port-name
spec:
  selector:
    app: metrics-app
  ports:
  - port: 9113  # No name!
    targetPort: 9113
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: no-port-monitor
  labels:
    team: frontend
spec:
  selector:
    matchLabels:
      app: no-port-name
  endpoints:
  - port: metrics  # Tries to find port named "metrics"
    interval: 30s
EOF
```

**Check Prometheus logs:**
```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=prometheus --tail=50 | grep -i "no-port-name"
```

Error: port "metrics" not found!

## Step 8: Test NetworkPolicy blocking scraping

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-monitoring
spec:
  podSelector:
    matchLabels:
      app: metrics-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: metrics-app
EOF
```

**Check Prometheus targets:**
```bash
# Targets show as DOWN - scrape fails
```

**Fix by allowing Prometheus:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
spec:
  podSelector:
    matchLabels:
      app: metrics-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
    ports:
    - protocol: TCP
      port: 9113
EOF
```

## Step 9: Test endpoint relabeling

```bash
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: relabel-monitor
  labels:
    team: frontend
spec:
  selector:
    matchLabels:
      app: metrics-app
  endpoints:
  - port: metrics
    interval: 30s
    relabelings:
    - sourceLabels: [__address__]
      targetLabel: instance
      replacement: 'my-custom-instance'
    - action: drop
      sourceLabels: [__meta_kubernetes_pod_name]
      regex: 'metrics-app-.*'  # Drops all targets!
EOF
```

**Check Prometheus:**
```bash
# All targets dropped by relabeling!
```

## Step 10: Test multiple endpoints

```bash
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: multi-endpoint-monitor
  labels:
    team: frontend
spec:
  selector:
    matchLabels:
      app: metrics-app
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
  - port: web
    interval: 30s
    path: /nginx-metrics
EOF
```

**Check targets:**
```bash
# Two scrape configs per service
```

## Step 11: Test namespace selector

```bash
kubectl create namespace production

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod-app
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prod-app
  template:
    metadata:
      labels:
        app: prod-app
    spec:
      containers:
      - name: app
        image: nginx
        ports:
        - name: metrics
          containerPort: 9113
---
apiVersion: v1
kind: Service
metadata:
  name: prod-app-svc
  namespace: production
  labels:
    app: prod-app
spec:
  selector:
    app: prod-app
  ports:
  - name: metrics
    port: 9113
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: prod-monitor
  namespace: production
  labels:
    team: frontend
spec:
  selector:
    matchLabels:
      app: prod-app
  endpoints:
  - port: metrics
EOF
```

**Prometheus can't see it!** (Different namespace)

**Fix Prometheus to watch all namespaces:**
```bash
kubectl patch prometheus prometheus -n monitoring --type=merge -p '{"spec":{"serviceMonitorNamespaceSelector":{}}}'
```

## Step 12: Check Prometheus configuration

```bash
# Get generated config
kubectl exec -n monitoring prometheus-prometheus-0 -- cat /etc/prometheus/config_out/prometheus.env.yaml | head -100
```

Shows all scrape configs generated from ServiceMonitors.

## Step 13: Test PodMonitor (alternative to ServiceMonitor)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: pod-monitor
  labels:
    team: frontend
spec:
  selector:
    matchLabels:
      app: metrics-app
  podMetricsEndpoints:
  - port: metrics
    interval: 30s
EOF
```

Scrapes pods directly (no service needed)!

## Step 14: Debug missing targets

```bash
# Check ServiceMonitor status
kubectl get servicemonitor -A
kubectl describe servicemonitor correct-label-monitor

# Check Prometheus logs
kubectl logs -n monitoring -l app.kubernetes.io/name=prometheus --tail=100

# Check if Prometheus can reach service
kubectl exec -n monitoring prometheus-prometheus-0 -- wget -O- http://metrics-app-svc.default.svc:9113/metrics 2>&1 || echo "Connection failed"
```

## Step 15: Test PrometheusRule (alerting)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: example-rules
  namespace: monitoring
  labels:
    prometheus: prometheus
spec:
  groups:
  - name: example
    interval: 30s
    rules:
    - alert: HighErrorRate
      expr: rate(http_requests_total{status="500"}[5m]) > 0.05
      for: 10m
      labels:
        severity: critical
      annotations:
        summary: "High error rate detected"
EOF
```

**Check rules in Prometheus UI:**
```bash
# Access http://localhost:9090/rules
```

## Key Observations

✅ **ServiceMonitor labels** - must match Prometheus selector
✅ **Port names** - Service port must be named, ServiceMonitor references by name
✅ **NetworkPolicy** - can block Prometheus scraping
✅ **Namespace selectors** - control which namespaces to monitor
✅ **Relabeling** - can accidentally drop all targets
✅ **PodMonitor** - alternative that scrapes pods directly

## Production Patterns

**Standard ServiceMonitor:**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app-monitor
  labels:
    prometheus: main
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
    scheme: http
```

**Cross-namespace monitoring:**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
spec:
  serviceMonitorSelector:
    matchLabels:
      prometheus: main
  serviceMonitorNamespaceSelector: {}  # All namespaces
  podMonitorSelector:
    matchLabels:
      prometheus: main
  podMonitorNamespaceSelector: {}
```

**NetworkPolicy allowing Prometheus:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus
spec:
  podSelector:
    matchLabels:
      app: myapp
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 9090
```

## Cleanup

```bash
pkill -f "port-forward.*prometheus"
kubectl delete namespace monitoring production
kubectl delete servicemonitor --all
kubectl delete podmonitor --all
kubectl delete deployment metrics-app
kubectl delete service metrics-app-svc no-port-name-svc
kubectl delete networkpolicy block-monitoring allow-monitoring
```

---
