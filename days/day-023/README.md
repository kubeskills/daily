## Day 23: CNI Plugin Failures - Network Isolation

## THE IDEA:
Simulate CNI plugin failures and watch pods get stuck in ContainerCreating with
"network not ready" errors. Explore how different CNI plugins fail and recover.

## THE SETUP:
Deploy pods, break the CNI plugin configuration, observe network assignment
failures, and learn to diagnose missing pod IPs and routes.

## WHAT I LEARNED:
- Pods without CNI stay in ContainerCreating forever
- Each pod needs an IP from the pod CIDR range
- CNI failures show as "failed to setup network" in kubelet logs
- Network policies only work with CNI plugins that support them

## WHY IT MATTERS:
CNI issues cause:
- New pods unable to start across the cluster
- Silent networking failures (pods run but can't communicate)
- Performance problems from misconfigured MTU

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-023

Tomorrow: kube-proxy modes and service routing failures.

---

# Killercoda Lab Instructions

## Step 1: Check current CNI plugin

```bash
# Check CNI configuration
ls -la /etc/cni/net.d/

# View CNI config
cat /etc/cni/net.d/*.conf* 2>/dev/null || echo "CNI config location varies"

# Check CNI binaries
ls -la /opt/cni/bin/ 2>/dev/null || ls -la /usr/lib/cni/ 2>/dev/null

```

## Step 2: Deploy baseline pods with networking

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nettest-1
  labels:
    app: nettest
spec:
  containers:
  - name: netshoot
    image: nicolaka/netshoot
    command: ['sleep', '3600']
---
apiVersion: v1
kind: Pod
metadata:
  name: nettest-2
  labels:
    app: nettest
spec:
  containers:
  - name: netshoot
    image: nicolaka/netshoot
    command: ['sleep', '3600']
EOF

```

**Check pod IPs:**

```bash
kubectl get pods -o wide

```

Both should have IPs assigned.

## Step 3: Test pod-to-pod connectivity

```bash
POD1_IP=$(kubectl get pod nettest-1 -o jsonpath='{.status.podIP}')
POD2_IP=$(kubectl get pod nettest-2 -o jsonpath='{.status.podIP}')

echo "Pod 1 IP: $POD1_IP"
echo "Pod 2 IP: $POD2_IP"

# Test connectivity
kubectl exec nettest-1 -- ping -c 3 $POD2_IP

```

Should succeed.

## Step 4: Check pod network namespace

```bash
kubectl exec nettest-1 -- ip addr show
kubectl exec nettest-1 -- ip route show

```

**Check interfaces:**

```bash
kubectl exec nettest-1 -- ip link show

```

Look for `eth0` interface with pod IP.

## Step 5: Simulate CNI configuration issue

**Note:** Breaking CNI in a live cluster is dangerous. We'll simulate issues.

```bash
# Create a pod with hostNetwork to bypass CNI
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: host-network-pod
spec:
  hostNetwork: true  # Uses host network, no CNI
  containers:
  - name: app
    image: nginx
EOF

```

**Check IP:**

```bash
kubectl get pod host-network-pod -o wide

```

IP matches node IP!

## Step 6: Test DNS resolution in pods

```bash
kubectl exec nettest-1 -- nslookup kubernetes.default
kubectl exec nettest-1 -- cat /etc/resolv.conf

```

**Check CoreDNS:**

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns

```

## Step 7: Inspect CNI plugin logs

```bash
# On node (if accessible)
# journalctl -u kubelet | grep -i cni

# Check kubelet logs in Killercoda
kubectl get nodes
kubectl describe node <node-name> | grep -i network

```

## Step 8: Test with dnsPolicy variations

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: dns-default
spec:
  dnsPolicy: Default  # Use node's DNS
  containers:
  - name: app
    image: busybox
    command: ['sleep', '3600']
---
apiVersion: v1
kind: Pod
metadata:
  name: dns-clusterfirst
spec:
  dnsPolicy: ClusterFirst  # Use cluster DNS (default)
  containers:
  - name: app
    image: busybox
    command: ['sleep', '3600']
---
apiVersion: v1
kind: Pod
metadata:
  name: dns-none
spec:
  dnsPolicy: None  # Manual DNS config
  dnsConfig:
    nameservers:
    - 8.8.8.8
    searches:
    - default.svc.cluster.local
  containers:
  - name: app
    image: busybox
    command: ['sleep', '3600']
EOF

```

**Compare DNS configs:**

```bash
kubectl exec dns-default -- cat /etc/resolv.conf
echo "---"
kubectl exec dns-clusterfirst -- cat /etc/resolv.conf
echo "---"
kubectl exec dns-none -- cat /etc/resolv.conf

```

## Step 9: Test MTU issues

```bash
# Check MTU on pod interface
kubectl exec nettest-1 -- ip link show eth0 | grep mtu

# Test with large packets
kubectl exec nettest-1 -- ping -M do -s 1400 -c 3 $POD2_IP
kubectl exec nettest-1 -- ping -M do -s 1472 -c 3 $POD2_IP

```

If MTU misconfigured, large packets fail.

## Step 10: Examine pod network from node perspective

```bash
# This requires node access - simulated in Killercoda
# On actual node:
# ip netns list
# nsenter -t <pid> -n ip addr

# Check CNI network bridges
# ip link show | grep -i cni
# brctl show (if using bridge CNI)

```

## Step 11: Test NetworkPolicy enforcement

```bash
# Check if NetworkPolicy is supported
kubectl get networkpolicies

# Create deny-all policy
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector:
    matchLabels:
      app: nettest
  policyTypes:
  - Ingress
  - Egress
EOF

```

**Test connectivity (should fail):**

```bash
kubectl exec nettest-1 -- ping -c 2 $POD2_IP 2>&1 || echo "Blocked by NetworkPolicy"

```

## Step 12: Check CNI plugin capabilities

```bash
# Different CNI plugins have different features:
# - Calico: NetworkPolicy, BGP routing, encryption
# - Cilium: eBPF, L7 policies, Hubble observability
# - Flannel: Simple overlay, no NetworkPolicy
# - Weave: Mesh network, encryption

# Check which CNI is installed
kubectl get pods -n kube-system -o wide | grep -E "calico|cilium|flannel|weave"

```

## Step 13: Simulate IP exhaustion

```bash
# Check pod CIDR
kubectl cluster-info dump | grep -m 1 cluster-cidr

# Deploy many pods to test CIDR limits (careful!)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ip-hungry
spec:
  replicas: 10
  selector:
    matchLabels:
      app: ip-hungry
  template:
    metadata:
      labels:
        app: ip-hungry
    spec:
      containers:
      - name: pause
        image: k8s.gcr.io/pause:3.9
EOF

```

**Check IP allocation:**

```bash
kubectl get pods -o wide | grep ip-hungry

```

## Step 14: Test service ClusterIP routing

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: nettest-svc
spec:
  selector:
    app: nettest
  ports:
  - port: 80
    targetPort: 80
EOF

```

**Test service connectivity:**

```bash
kubectl exec nettest-1 -- curl -m 5 nettest-svc 2>&1 || echo "No web server running"

```

## Step 15: Diagnose CNI failures

```bash
# Create pod and watch for CNI errors
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cni-debug
spec:
  containers:
  - name: app
    image: nginx
EOF

```

**Check for CNI issues:**

```bash
kubectl describe pod cni-debug | grep -A 10 Events
kubectl get pod cni-debug -o jsonpath='{.status.conditions[?(@.type=="Ready")].message}'

```

**Common errors:**

- "network not ready: container runtime network not ready"
- "failed to setup network for sandbox"
- "failed to allocate IP address"

## Key Observations

✅ **CNI assigns pod IPs** - without CNI, pods stuck in ContainerCreating
✅ **Network policies** - require CNI plugin support
✅ **DNS integration** - CNI must configure DNS properly
✅ **MTU matters** - mismatched MTU causes packet loss
✅ **Pod CIDR exhaustion** - causes IP allocation failures

## Production Patterns

**Check CNI health:**

```bash
# Verify CNI pods running
kubectl get pods -n kube-system -l k8s-app=cilium  # or calico-node
kubectl logs -n kube-system <cni-pod-name>

```

**Test network connectivity:**

```bash
# Deploy test pods in different nodes
kubectl run test-source --image=nicolaka/netshoot --command -- sleep 3600
kubectl run test-dest --image=nginx

# Test connectivity
kubectl exec test-source -- curl <pod-ip>

```

**Monitor CNI performance:**

```bash
# Check for CNI errors in logs
kubectl logs -n kube-system -l app=calico-node --tail=100 | grep -i error

```

**Backup CNI configuration:**

```bash
# Before changes, backup CNI config
kubectl get -n kube-system configmap cni-config -o yaml > cni-backup.yaml

```

## Cleanup

```bash
kubectl delete networkpolicy deny-all 2>/dev/null
kubectl delete pod nettest-1 nettest-2 host-network-pod dns-default dns-clusterfirst dns-none cni-debug 2>/dev/null
kubectl delete deployment ip-hungry 2>/dev/null
kubectl delete service nettest-svc 2>/dev/null

```

---

Next: Day 24 - kube-proxy Modes and Service Routing