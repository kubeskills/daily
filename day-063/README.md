## Day 63: CoreDNS Failures - When DNS Dies

### Email Copy

**Subject:** Day 63: CoreDNS Failures - When Nothing Resolves

THE IDEA:
Break CoreDNS and watch all service discovery fail. Pods can't resolve service names,
external DNS queries fail, and applications crash trying to connect to databases.

THE SETUP:
Corrupt CoreDNS ConfigMap, exhaust CoreDNS pods, create DNS loops, and test what
happens when the cluster DNS service goes down.

WHAT I LEARNED:
- CoreDNS resolves service names to ClusterIPs
- /etc/resolv.conf in pods points to CoreDNS
- ndots:5 causes queries for external domains to try cluster suffixes first
- CoreDNS forwards external queries to upstream DNS
- DNS loops can occur with misconfigured forwarders

WHY IT MATTERS:
DNS failures cause:
- Services unreachable by name (must use IPs)
- External API calls fail (can't resolve public domains)
- Database connections fail (hostname not resolved)
- Intermittent failures hard to debug

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-063

Tomorrow: Persistent volume provisioning failures.

---
Chad


---

## Killercoda Lab Instructions



## Step 1: Check CoreDNS status

```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS service
kubectl get svc -n kube-system kube-dns

# Get DNS service IP
DNS_IP=$(kubectl get svc -n kube-system kube-dns -o jsonpath='{.spec.clusterIP}')
echo "DNS Service IP: $DNS_IP"
```

## Step 2: Test normal DNS resolution

```bash
# Deploy test pod
kubectl run dns-test --image=busybox --restart=Never --command -- sleep 3600

kubectl wait --for=condition=Ready pod dns-test --timeout=60s

# Check resolv.conf
kubectl exec dns-test -- cat /etc/resolv.conf

# Test service DNS
kubectl create deployment nginx --image=nginx --replicas=2
kubectl expose deployment nginx --port=80

kubectl exec dns-test -- nslookup nginx.default.svc.cluster.local
```

## Step 3: Check CoreDNS configuration

```bash
# Get CoreDNS ConfigMap
kubectl get configmap -n kube-system coredns -o yaml

# Show Corefile
kubectl get configmap -n kube-system coredns -o jsonpath='{.data.Corefile}'
echo ""
```

## Step 4: Simulate CoreDNS pod failure

```bash
# Scale CoreDNS to 0
kubectl scale deployment -n kube-system coredns --replicas=0

# Wait for pods to terminate
sleep 10

# Try DNS resolution (will fail)
kubectl exec dns-test -- nslookup nginx.default.svc.cluster.local 2>&1 || echo "DNS resolution failed!"

# Check timeout
time kubectl exec dns-test -- nslookup nginx.default.svc.cluster.local 2>&1 || echo "DNS timeout"
```

## Step 5: Restore CoreDNS

```bash
# Scale back up
kubectl scale deployment -n kube-system coredns --replicas=2

kubectl wait --for=condition=Ready pod -n kube-system -l k8s-app=kube-dns --timeout=60s

# Test resolution works again
kubectl exec dns-test -- nslookup nginx.default.svc.cluster.local
```

## Step 6: Test CoreDNS under load

```bash
# Create many DNS queries
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dns-load
spec:
  replicas: 10
  selector:
    matchLabels:
      app: dns-load
  template:
    metadata:
      labels:
        app: dns-load
    spec:
      containers:
      - name: load
        image: busybox
        command:
        - sh
        - -c
        - |
          while true; do
            nslookup nginx.default.svc.cluster.local
            nslookup google.com
            sleep 0.1
          done
EOF

kubectl wait --for=condition=Ready pod -l app=dns-load --timeout=60s

# Check CoreDNS load
sleep 10
kubectl top pod -n kube-system -l k8s-app=kube-dns
```

## Step 7: Corrupt CoreDNS configuration

```bash
# Backup current config
kubectl get configmap -n kube-system coredns -o yaml > /tmp/coredns-backup.yaml

# Create broken configuration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf INVALID_SYNTAX_HERE
        cache 30
        loop
        reload
        loadbalance
    }
EOF

# Restart CoreDNS to pick up config
kubectl rollout restart deployment -n kube-system coredns

# Wait and check status
sleep 15
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

## Step 8: Check CoreDNS logs for errors

```bash
# Check logs for configuration errors
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50 | grep -i "error\|fail\|invalid"
```

## Step 9: Restore working configuration

```bash
# Restore from backup
kubectl apply -f /tmp/coredns-backup.yaml

# Restart CoreDNS
kubectl rollout restart deployment -n kube-system coredns

kubectl wait --for=condition=Ready pod -n kube-system -l k8s-app=kube-dns --timeout=60s

# Verify DNS works
kubectl exec dns-test -- nslookup kubernetes.default.svc.cluster.local
```

## Step 10: Test DNS loop

```bash
# Create loop by forwarding to itself
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
        }
        forward . 127.0.0.1:53
        cache 30
        loop
        reload
    }
EOF

kubectl rollout restart deployment -n kube-system coredns
sleep 15

# Check for loop detection
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=20 | grep -i loop
```

## Step 11: Test ndots behavior

```bash
# Restore working config
kubectl apply -f /tmp/coredns-backup.yaml
kubectl rollout restart deployment -n kube-system coredns
kubectl wait --for=condition=Ready pod -n kube-system -l k8s-app=kube-dns --timeout=60s

# Check default ndots (usually 5)
kubectl exec dns-test -- cat /etc/resolv.conf | grep ndots

# With ndots:5, "google.com" tries these first:
# - google.com.default.svc.cluster.local
# - google.com.svc.cluster.local
# - google.com.cluster.local
# - google.com. (finally succeeds)

# Show the query pattern
kubectl exec dns-test -- sh -c 'nslookup google.com 2>&1 | grep -E "Server:|Name:"'
```

## Step 12: Test external DNS forwarding failure

```bash
# Create config that breaks external DNS
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
        }
        forward . 192.0.2.1
        cache 30
        loop
        reload
    }
EOF

kubectl rollout restart deployment -n kube-system coredns
kubectl wait --for=condition=Ready pod -n kube-system -l k8s-app=kube-dns --timeout=60s

# Internal DNS works
kubectl exec dns-test -- nslookup nginx.default.svc.cluster.local

# External DNS fails (upstream unreachable)
kubectl exec dns-test -- nslookup google.com 2>&1 || echo "External DNS failed"
```

## Step 13: Test DNS caching

```bash
# Restore working config
kubectl apply -f /tmp/coredns-backup.yaml
kubectl rollout restart deployment -n kube-system coredns
kubectl wait --for=condition=Ready pod -n kube-system -l k8s-app=kube-dns --timeout=60s

# Query same domain multiple times
for i in {1..5}; do
  time kubectl exec dns-test -- nslookup google.com 2>&1 | grep "Server:"
done

# First query slower (cache miss), subsequent faster (cache hit)
```

## Step 14: Test split-horizon DNS

```bash
# Test that cluster DNS takes precedence
kubectl exec dns-test -- nslookup kubernetes.default

# vs external (if kubernetes.com existed)
kubectl exec dns-test -- nslookup kubernetes.com
```

## Step 15: Diagnose DNS issues

```bash
cat > /tmp/dns-diagnosis.sh << 'EOF'
#!/bin/bash
echo "=== DNS Diagnosis Report ==="

echo -e "\n1. CoreDNS Pod Status:"
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide

echo -e "\n2. CoreDNS Service:"
kubectl get svc -n kube-system kube-dns

echo -e "\n3. CoreDNS Configuration:"
kubectl get configmap -n kube-system coredns -o jsonpath='{.data.Corefile}' | head -20

echo -e "\n4. CoreDNS Logs (recent errors):"
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=20 | grep -i "error\|fail" || echo "No errors found"

echo -e "\n5. Test DNS Resolution:"
TEST_POD=$(kubectl get pods -o name | head -1 | cut -d'/' -f2)
if [ -n "$TEST_POD" ]; then
  echo "Testing from pod: $TEST_POD"
  kubectl exec $TEST_POD -- nslookup kubernetes.default 2>&1 | head -5
else
  echo "No test pod available"
fi

echo -e "\n6. DNS Query Performance:"
kubectl exec dns-test -- sh -c 'time nslookup kubernetes.default' 2>&1 | grep real

echo -e "\n7. CoreDNS Resource Usage:"
kubectl top pod -n kube-system -l k8s-app=kube-dns 2>/dev/null || echo "Metrics not available"

echo -e "\n8. Common DNS Issues:"
echo "   - CoreDNS pods down: All DNS fails"
echo "   - Corrupted config: CoreDNS crashes"
echo "   - Forward loop: Queries never resolve"
echo "   - Upstream unreachable: External DNS fails"
echo "   - High ndots: Many unnecessary queries"
echo "   - Cache issues: Stale DNS records"
EOF

chmod +x /tmp/dns-diagnosis.sh
/tmp/dns-diagnosis.sh
```

## Key Observations

✅ **CoreDNS critical** - all service discovery depends on it
✅ **resolv.conf** - pods configured to use CoreDNS IP
✅ **ndots:5** - causes multiple query attempts
✅ **Forward directive** - sends external queries upstream
✅ **DNS loops** - detected by CoreDNS loop plugin
✅ **Cache** - improves performance but can serve stale data

## Production Patterns

**CoreDNS high-availability:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
spec:
  replicas: 3  # At least 2 for HA
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  k8s-app: kube-dns
              topologyKey: kubernetes.io/hostname
      priorityClassName: system-cluster-critical
      containers:
      - name: coredns
        image: coredns/coredns:1.10.1
        resources:
          requests:
            cpu: 100m
            memory: 70Mi
          limits:
            cpu: 500m
            memory: 170Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: 8181
```

**Optimized Corefile:**
```
.:53 {
    errors
    health {
        lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf {
        max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance round_robin
}
```

**Custom pod DNS config (reduce ndots):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: optimized-dns
spec:
  dnsConfig:
    options:
    - name: ndots
      value: "1"  # Reduce from default 5
    - name: timeout
      value: "2"
    - name: attempts
      value: "2"
  containers:
  - name: app
    image: myapp:latest
```

**CoreDNS monitoring:**
```yaml
# Prometheus alerts
- alert: CoreDNSDown
  expr: up{job="kube-dns"} == 0
  for: 5m
  annotations:
    summary: "CoreDNS is down"

- alert: CoreDNSHighLatency
  expr: |
    histogram_quantile(0.99,
      rate(coredns_dns_request_duration_seconds_bucket[5m])
    ) > 0.5
  annotations:
    summary: "CoreDNS latency > 500ms"

- alert: CoreDNSHighErrorRate
  expr: |
    rate(coredns_dns_responses_total{rcode="SERVFAIL"}[5m]) > 10
  annotations:
    summary: "CoreDNS SERVFAIL rate high"
```

**DNS debugging pod:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dnsutils
spec:
  containers:
  - name: dnsutils
    image: registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3
    command:
      - sleep
      - "infinity"
```

## Cleanup

```bash
kubectl delete deployment dns-load nginx
kubectl delete pod dns-test
kubectl delete service nginx
kubectl apply -f /tmp/coredns-backup.yaml
kubectl rollout restart deployment -n kube-system coredns
rm -f /tmp/dns-diagnosis.sh /tmp/coredns-backup.yaml
```

---
Next: Day 64 - PV Provisioning Failures and Storage Chaos
