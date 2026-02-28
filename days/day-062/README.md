## Day 62: CNI Plugin Failures - Networking Goes Dark

### Email Copy

**Subject:** Day 62: CNI Plugin Failures - When Pods Can't Communicate

THE IDEA:
Break the CNI plugin and watch all pod networking fail. Pods start but can't reach
each other, DNS fails, and new pods get stuck in ContainerCreating with network errors.

THE SETUP:
Corrupt CNI configuration files, exhaust IP address pools, break pod CIDR routing,
and test what happens when the network plugin crashes.

WHAT I LEARNED:
- CNI plugins run on every node (DaemonSet)
- Each pod gets IP from pod CIDR range
- CNI config in /etc/cni/net.d/ on nodes
- IP address exhaustion prevents new pod scheduling
- Network policies depend on CNI plugin support

WHY IT MATTERS:
CNI failures cause:
- All new pods stuck in ContainerCreating
- Existing pods lose network connectivity
- DNS resolution completely broken
- Cluster unusable despite nodes being Ready

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-062

Tomorrow: CoreDNS failures and DNS resolution chaos.

---
Chad
Kube Daily - Week 10: Network & Storage Deep Dive


---

## Killercoda Lab Instructions



## Step 1: Check current CNI plugin

```bash
# Check CNI configuration
ls -la /etc/cni/net.d/ 2>/dev/null || echo "CNI config not accessible"

# Check CNI pods (usually Calico, Flannel, Cilium, etc.)
kubectl get pods -n kube-system | grep -E "calico|flannel|cilium|weave|canal"

# Get pod CIDR
kubectl cluster-info dump | grep -i cidr
```

## Step 2: Check current pod networking

```bash
# Deploy test pods
kubectl run test-pod-1 --image=nginx
kubectl run test-pod-2 --image=nginx

kubectl wait --for=condition=Ready pod test-pod-1 test-pod-2 --timeout=60s

# Get pod IPs
POD1_IP=$(kubectl get pod test-pod-1 -o jsonpath='{.status.podIP}')
POD2_IP=$(kubectl get pod test-pod-2 -o jsonpath='{.status.podIP}')

echo "Pod 1 IP: $POD1_IP"
echo "Pod 2 IP: $POD2_IP"

# Test connectivity
kubectl exec test-pod-1 -- ping -c 3 $POD2_IP
```

## Step 3: Check CNI binary and config

```bash
# Check CNI binaries
ls -la /opt/cni/bin/ 2>/dev/null || echo "CNI binaries not accessible"

# Check CNI config files
cat /etc/cni/net.d/*.conf* 2>/dev/null || echo "CNI config not accessible"
```

## Step 4: Simulate CNI config corruption

```bash
# Create pod that will fail due to CNI issues
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: network-test
  labels:
    test: cni
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: 10m
        memory: 16Mi
EOF

# Check status
sleep 5
kubectl get pod network-test
kubectl describe pod network-test | grep -A 10 "Events:"
```

## Step 5: Test IP address exhaustion

```bash
# Check pod CIDR range
kubectl get nodes -o jsonpath='{.items[0].spec.podCIDR}'
echo ""

# Calculate available IPs (for demonstration)
echo "Pod CIDR typically provides 254 IPs per /24 network"
echo "Creating many pods to exhaust IP pool..."

# Create deployment with many replicas
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ip-exhaustion
spec:
  replicas: 50
  selector:
    matchLabels:
      app: exhaust
  template:
    metadata:
      labels:
        app: exhaust
    spec:
      containers:
      - name: app
        image: nginx
        resources:
          requests:
            cpu: 10m
            memory: 16Mi
EOF

# Watch pod creation
kubectl get pods -l app=exhaust -w
```

## Step 6: Check for failed pod networking

```bash
# Look for pods stuck in ContainerCreating
kubectl get pods -A --field-selector status.phase=Pending

# Check pod events for network errors
for pod in $(kubectl get pods -l app=exhaust -o name | head -5); do
  echo "Checking $pod:"
  kubectl describe $pod | grep -A 5 "Events:" | grep -i "network\|cni"
done
```

## Step 7: Test network policy with CNI support

```bash
# Check if CNI supports NetworkPolicy
kubectl api-resources | grep networkpolicies

# Create test namespace
kubectl create namespace netpol-test

# Deploy test pods
kubectl run web -n netpol-test --image=nginx --port=80
kubectl run client -n netpol-test --image=busybox --command -- sleep 3600

kubectl wait --for=condition=Ready pod -n netpol-test --all --timeout=60s

# Test connectivity before policy
WEB_IP=$(kubectl get pod web -n netpol-test -o jsonpath='{.status.podIP}')
kubectl exec -n netpol-test client -- wget -O- --timeout=2 http://$WEB_IP 2>&1 | grep -i "welcome\|connected" || echo "Connection failed"
```

## Step 8: Apply network policy (requires CNI support)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: netpol-test
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

# Test connectivity after policy (should fail)
sleep 5
kubectl exec -n netpol-test client -- wget -O- --timeout=2 http://$WEB_IP 2>&1 || echo "Blocked by NetworkPolicy"
```

## Step 9: Check CNI plugin logs

```bash
# Find CNI plugin pods
CNI_PODS=$(kubectl get pods -n kube-system -l k8s-app=kube-proxy -o name 2>/dev/null || \
           kubectl get pods -n kube-system | grep -E "calico|flannel|cilium" | awk '{print $1}')

if [ -n "$CNI_PODS" ]; then
  echo "CNI plugin pods found"
  for pod in $CNI_PODS; do
    echo "Logs from $pod:"
    kubectl logs -n kube-system $pod --tail=20 2>/dev/null | grep -i "error\|fail\|warn" || echo "No errors found"
  done
else
  echo "CNI plugin pods not found or not accessible"
fi
```

## Step 10: Test pod-to-pod connectivity across nodes

```bash
# Get node count
NODE_COUNT=$(kubectl get nodes --no-headers | wc -l)

if [ $NODE_COUNT -gt 1 ]; then
  echo "Multi-node cluster detected"

  # Create pod on each node with nodeSelector
  kubectl label nodes $(kubectl get nodes -o name | head -1 | cut -d'/' -f2) test-node=node1 --overwrite

  kubectl run cross-node-1 --image=nginx --overrides='{"spec":{"nodeSelector":{"test-node":"node1"}}}'

  kubectl wait --for=condition=Ready pod cross-node-1 --timeout=60s

  CROSS_IP=$(kubectl get pod cross-node-1 -o jsonpath='{.status.podIP}')

  # Test from pod on different node
  kubectl exec test-pod-1 -- ping -c 3 $CROSS_IP || echo "Cross-node connectivity issue"
else
  echo "Single-node cluster, skipping cross-node test"
fi
```

## Step 11: Check routing tables

```bash
# Check if we can access node routing
echo "Checking pod routing (requires node access):"
kubectl run debug-pod --rm -it --image=nicolaka/netshoot --restart=Never -- route -n 2>/dev/null || echo "Route table not accessible from pod"
```

## Step 12: Test DNS resolution (depends on CNI)

```bash
# DNS should work with functioning CNI
kubectl run dns-test --rm -it --image=busybox --restart=Never -- nslookup kubernetes.default.svc.cluster.local

# Test from existing pod
kubectl exec test-pod-1 -- nslookup kubernetes.default 2>&1 || echo "DNS resolution failed"
```

## Step 13: Check for CNI-related errors in kubelet

```bash
# Check kubelet logs for CNI errors
echo "Checking for CNI errors in kubelet logs:"
echo "(In real clusters, you would check: journalctl -u kubelet | grep CNI)"
echo "Common CNI errors:"
echo "- failed to set up sandbox container network"
echo "- failed to find plugin in path"
echo "- failed to allocate for range"
```

## Step 14: Test service networking

```bash
# Create service
kubectl expose pod test-pod-1 --port=80 --name=test-svc

# Get service IP
SVC_IP=$(kubectl get svc test-svc -o jsonpath='{.spec.clusterIP}')
echo "Service IP: $SVC_IP"

# Test service access
kubectl exec test-pod-2 -- curl -s -m 3 http://$SVC_IP 2>&1 | head -5 || echo "Service networking issue"
```

## Step 15: Diagnose CNI issues

```bash
cat > /tmp/cni-diagnosis.sh << 'EOF'
#!/bin/bash
echo "=== CNI Diagnosis Report ==="

echo -e "\n1. Pod Networking Status:"
kubectl get pods -A -o wide | grep -E "ContainerCreating|Error|CrashLoopBackOff" | head -10

echo -e "\n2. Pod CIDR Configuration:"
kubectl get nodes -o custom-columns=NAME:.metadata.name,CIDR:.spec.podCIDR

echo -e "\n3. CNI Pods Status:"
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide 2>/dev/null || echo "kube-proxy pods not found"

echo -e "\n4. Network Policy Resources:"
kubectl get networkpolicies -A

echo -e "\n5. Service CIDR:"
kubectl cluster-info dump | grep -m 1 service-cluster-ip-range || echo "Service CIDR not found"

echo -e "\n6. Recent Pod Events (network-related):"
kubectl get events -A --sort-by='.lastTimestamp' | grep -i "network\|cni\|sandbox" | tail -10

echo -e "\n7. Pod IP Assignment:"
kubectl get pods -A -o custom-columns=NAME:.metadata.name,NAMESPACE:.metadata.namespace,IP:.status.podIP,NODE:.spec.nodeName | grep -v "IP.*<none>" | head -10

echo -e "\n8. Common CNI Issues:"
echo "   - IP exhaustion: Too many pods for pod CIDR"
echo "   - CNI plugin crash: Pods stuck in ContainerCreating"
echo "   - Config corruption: Network setup fails"
echo "   - Cross-node routing: Pods can't reach other nodes"
echo "   - NetworkPolicy: Requires CNI plugin support"
EOF

chmod +x /tmp/cni-diagnosis.sh
/tmp/cni-diagnosis.sh
```

## Key Observations

✅ **CNI required** - all pod networking depends on CNI plugin
✅ **Pod CIDR** - each node allocates IPs from this range
✅ **IP exhaustion** - blocks new pod creation
✅ **ContainerCreating** - common symptom of CNI failure
✅ **Network policies** - require CNI plugin support
✅ **Cross-node traffic** - requires proper routing setup

## Production Patterns

**CNI plugin monitoring:**
```yaml
# Prometheus alerts
- alert: CNIPluginDown
  expr: up{job="kube-system/calico-node"} == 0
  for: 5m
  annotations:
    summary: "CNI plugin on {{ $labels.node }} is down"

- alert: PodCIDRExhaustion
  expr: |
    (count(kube_pod_info) by (node) /
     on(node) group_left() node_network_address_size) > 0.9
  annotations:
    summary: "Pod CIDR on {{ $labels.node }} is 90% exhausted"

- alert: PodsStuckInContainerCreating
  expr: |
    count(kube_pod_status_phase{phase="Pending"}) by (namespace) > 5
  for: 10m
  annotations:
    summary: "{{ $value }} pods stuck in Pending/ContainerCreating"
```

**CNI health check script:**
```bash
#!/bin/bash
# cni-health-check.sh

echo "Checking CNI health..."

# Check CNI pods running
CNI_PODS=$(kubectl get pods -n kube-system -l k8s-app=calico-node --no-headers 2>/dev/null | wc -l)
NODE_COUNT=$(kubectl get nodes --no-headers | wc -l)

if [ "$CNI_PODS" -lt "$NODE_COUNT" ]; then
  echo "CRITICAL: CNI pods ($CNI_PODS) < nodes ($NODE_COUNT)"
  exit 2
fi

# Check for pending pods with network issues
STUCK_PODS=$(kubectl get pods -A --field-selector status.phase=Pending -o json | \
  jq '[.items[] | select(.status.conditions[]? | .reason == "ContainerCreating")] | length')

if [ "$STUCK_PODS" -gt 5 ]; then
  echo "WARNING: $STUCK_PODS pods stuck in ContainerCreating"
  exit 1
fi

echo "OK: CNI health check passed"
exit 0
```

**Pod CIDR expansion:**
```yaml
# For clusters running out of IPs, expand pod CIDR
# This requires cluster recreation or careful migration

# kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
networking:
  podSubnet: "10.244.0.0/16"  # Larger range
  serviceSubnet: "10.96.0.0/12"
```

**Network policy defaults:**
```yaml
# Default deny all in production namespaces
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Allow DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

## Cleanup

```bash
kubectl delete deployment ip-exhaustion
kubectl delete pod test-pod-1 test-pod-2 network-test cross-node-1 2>/dev/null
kubectl delete service test-svc
kubectl delete namespace netpol-test
rm -f /tmp/cni-diagnosis.sh
```

---
Next: Day 63 - CoreDNS Failures and DNS Resolution Chaos
