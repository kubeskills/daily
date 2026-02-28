## Day 24: kube-proxy Modes - Service Routing Breakdown

## THE IDEA:
Explore different kube-proxy modes (iptables, ipvs, userspace) and discover how
service routing breaks when proxy is misconfigured or failing.

## THE SETUP:
Deploy services, check kube-proxy mode, simulate proxy failures, and observe
how service ClusterIPs become unreachable despite healthy backend pods.

## WHAT I LEARNED:
- iptables mode creates thousands of rules (scales poorly)
- IPVS mode uses kernel load balancing (better performance)
- kube-proxy must run on every node for services to work
- Service ClusterIP is virtual - requires proxy to route traffic

## WHY IT MATTERS:
kube-proxy issues cause:
- Services unreachable despite healthy pods
- Random connection failures to some endpoints
- Performance degradation as services scale

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-024

Tomorrow: etcd performance impacting cluster stability.

---

# Killercoda Lab Instructions


## Step 1: Check current kube-proxy mode

```bash
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode

```

**Check kube-proxy pods:**

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=20

```

## Step 2: Deploy backend application

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: hashicorp/http-echo
        args:
        - "-text=Backend Pod: \\$(HOSTNAME)"
        - "-listen=:8080"
        env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
EOF

```

**Get service details:**

```bash
kubectl get service web-service
kubectl get endpoints web-service

```

## Step 3: Test service from within cluster

```bash
kubectl run test-client --rm -it --restart=Never --image=curlimages/curl -- sh -c '
for i in $(seq 1 10); do
  curl -s <http://web-service>
  sleep 0.5
done
'

```

Should see round-robin load balancing across pods.

## Step 4: Examine iptables rules (iptables mode)

```bash
# On node (if accessible):
# sudo iptables-save | grep web-service

# In Killercoda, check from privileged pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: iptables-viewer
spec:
  hostNetwork: true
  containers:
  - name: viewer
    image: nicolaka/netshoot
    command: ['sleep', '3600']
    securityContext:
      privileged: true
EOF

```

**View iptables:**

```bash
kubectl exec iptables-viewer -- iptables-save | grep -A 10 web-service

```

Look for:

- KUBE-SERVICES chain
- KUBE-SVC-* chain (service)
- KUBE-SEP-* chains (endpoints)

## Step 5: Count iptables rules

```bash
kubectl exec iptables-viewer -- iptables-save | wc -l

```

In large clusters: 10,000+ rules! Scales O(n) with services.

## Step 6: Test with multiple services

```bash
# Create 10 services
for i in $(seq 1 10); do
  kubectl create deployment test-$i --image=nginx --port=80
  kubectl expose deployment test-$i --port=80
done

# Count rules again
kubectl exec iptables-viewer -- iptables-save | wc -l

```

Rules multiply quickly!

## Step 7: Check service IP allocation

```bash
kubectl get services -A -o wide | head -20

# Check service CIDR
kubectl cluster-info dump | grep -m 1 service-cluster-ip-range

```

Services get IPs from this range.

## Step 8: Test service without endpoints

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: no-endpoints-svc
spec:
  selector:
    app: nonexistent  # No pods match!
  ports:
  - port: 80
    targetPort: 8080
EOF

```

**Check endpoints:**

```bash
kubectl get endpoints no-endpoints-svc

```

Shows `<none>` - service has no backends.

**Test connection:**

```bash
kubectl run test --rm -it --restart=Never --image=curlimages/curl -- curl -m 5 <http://no-endpoints-svc>

```

Timeout! Service IP exists but has no backends.

## Step 9: Test NodePort service

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
EOF

```

**Get node IP:**

```bash
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
echo "Node IP: $NODE_IP"

```

**Test from outside:**

```bash
kubectl run external-test --rm -it --restart=Never --image=curlimages/curl -- curl -m 5 http://$NODE_IP:30080

```

## Step 10: Simulate kube-proxy failure

```bash
# Scale kube-proxy to 0 (dangerous!)
kubectl scale daemonset kube-proxy -n kube-system --replicas=0 2>&1 || \\
echo "DaemonSet replicas can't be scaled directly"

# Alternative: Delete one kube-proxy pod
PROXY_POD=$(kubectl get pods -n kube-system -l k8s-app=kube-proxy -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod -n kube-system $PROXY_POD

# Wait for recreation
sleep 10

```

**During downtime, test service:**

```bash
kubectl run failtest --rm -it --restart=Never --image=curlimages/curl -- curl -m 5 <http://web-service> 2>&1 || echo "Service unavailable!"

```

## Step 11: Check kube-proxy sync errors

```bash
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=50 | grep -i error

```

Common errors:

- "Failed to sync iptables rules"
- "Failed to list endpoints"
- "Error getting node"

## Step 12: Test session affinity

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: sticky-service
spec:
  selector:
    app: web
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 300
  ports:
  - port: 80
    targetPort: 8080
EOF

```

**Test sticky sessions:**

```bash
kubectl run sticky-test --rm -it --restart=Never --image=curlimages/curl -- sh -c '
for i in $(seq 1 10); do
  curl -s <http://sticky-service>
  sleep 0.5
done
'

```

Same pod every time!

## Step 13: Test externalTrafficPolicy

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web-external
spec:
  type: NodePort
  externalTrafficPolicy: Local  # Only local pods
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30081
EOF

```

With `Local`, traffic only goes to pods on the receiving node.

## Step 14: Check service type differences

```bash
# ClusterIP (internal only)
kubectl get svc web-service

# NodePort (external access via node IP)
kubectl get svc web-nodeport

# LoadBalancer (cloud load balancer)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web-lb
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
EOF

```

**Note:** LoadBalancer pending in Killercoda (no cloud provider)

## Step 15: Debug service routing

```bash
# Check service selectors
kubectl describe service web-service | grep -A 3 Selector

# Check matching pods
kubectl get pods -l app=web

# Verify endpoints
kubectl get endpoints web-service -o yaml

# Test direct pod IP
POD_IP=$(kubectl get pod -l app=web -o jsonpath='{.items[0].status.podIP}')
kubectl run direct-test --rm -it --restart=Never --image=curlimages/curl -- curl http://$POD_IP:8080

```

## Key Observations

✅ **kube-proxy modes** - iptables (default), ipvs (better scale), userspace (legacy)
✅ **Service ClusterIP** - virtual IP, requires proxy for routing
✅ **Endpoints** - service with no endpoints = unreachable
✅ **iptables rules** - scale O(n), can cause performance issues
✅ **Session affinity** - ClientIP for sticky sessions

## Production Patterns

**Check kube-proxy health:**

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=100

```

**Monitor service endpoints:**

```bash
# Alert when service has no endpoints
kubectl get endpoints -A | awk '$2 == "<none>" {print $1, $2}'

```

**Use IPVS for large clusters:**

```yaml
# kube-proxy config
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  scheduler: "rr"  # round-robin

```

**ExternalTrafficPolicy for source IP:**

```yaml
apiVersion: v1
kind: Service
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local  # Preserves source IP

```

## Cleanup

```bash
kubectl delete deployment web-backend test-{1..10} 2>/dev/null
kubectl delete service web-service no-endpoints-svc web-nodeport sticky-service web-external web-lb test-{1..10} 2>/dev/null
kubectl delete pod iptables-viewer 2>/dev/null

```

---

Next: Day 25 - etcd Performance and Cluster Stability