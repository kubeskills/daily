# Day 4: Network Policies - Lock Yourself Out

## THE IDEA:
Deploy a deny-all network policy and watch all pod-to-pod communication break.
Then selectively re-enable traffic. Bonus: accidentally block DNS and debug it.

## THE SETUP:
Start with two pods that can talk. Apply a default-deny policy. Watch them fail.
Then create allow rules using label selectors.

## WHAT I LEARNED:
- Network policies are stateful (return traffic is automatically allowed)
- DNS breaks first (needs explicit allow to kube-system)
- Policies are additive (multiple policies = union of rules)
- Empty podSelector {} = applies to all pods in namespace

## WHY IT MATTERS:
One misconfigured network policy can:
- Break DNS cluster-wide
- Block health checks, causing pod evictions
- Create silent failures (connections timeout, no errors)

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-004

---


Check if your cluster has a CNI that supports Network Policies:

### Killercoda Lab Instructions

Check if your cluster has a CNI that supports Network Policies:


```
kubectl get pods -n kube-system | grep -E "calico|cilium|weave"

```

If using Kilercoda’s default, install Calico:

```bash
kubectl apply -f <https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml>

```

Wait ~30 seconds for Calico pods to be Ready.

**Step 1: Deploy two communicating pods**

```bash
# Pod 1: Web server
kubectl run web --image=nginx --labels="app=web" --port=80

# Pod 2: Client
kubectl run client --image=busybox --command -- sleep 3600

```

**Test connectivity:**

```bash
kubectl exec client -- wget -qO- --timeout=2 <http://web>

```

You should see the nginx welcome page HTML.

**Step 2: Apply a default-deny policy**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}  # Applies to all pods in namespace
  policyTypes:
  - Ingress
  - Egress
EOF

```

**Test connectivity again:**

```bash
kubectl exec client -- wget -qO- --timeout=2 <http://web>

```

Times out! All traffic is blocked.

**Step 3: Try to resolve DNS**

```bash
kubectl exec client -- nslookup kubernetes.default

```

Also fails! DNS is blocked (it’s egress traffic to kube-system).

**Step 4: Allow DNS**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
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
EOF

```

**Verify DNS works:**

```bash
kubectl exec client -- nslookup kubernetes.default

```

Success! But web access still blocked.

**Step 5: Allow traffic to web pods**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-ingress
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          run: client
    ports:
    - protocol: TCP
      port: 80
EOF

```

**Test again:**

```bash
kubectl exec client -- wget -qO- --timeout=2 <http://web>

```

Still fails! Why? Client can’t make egress connections.

**Step 6: Allow egress from client**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-client-egress
spec:
  podSelector:
    matchLabels:
      run: client
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: web
    ports:
    - protocol: TCP
      port: 80
  - to:  # Don't forget DNS!
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
EOF

```

**Final test:**

```bash
kubectl exec client -- wget -qO- --timeout=2 <http://web>

```

Success! 🎉

**Step 7: View all active policies**

```bash
kubectl get networkpolicies
kubectl describe networkpolicy allow-web-ingress

```

**Key Observations**

✅ **Default deny is namespace-scoped** - doesn’t affect other namespaces
✅ **DNS must be explicitly allowed** - most common mistake
✅ **Policies are additive** - multiple matching policies = union of rules
✅ **Ingress + Egress both needed** - connection requires both directions

**Common Production Patterns**

**Allow DNS everywhere:**

```yaml
egress:
- to:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: kube-system
  ports:
  - protocol: UDP
    port: 53

```

**Allow health checks from Ingress controller:**

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        name: ingress-nginx

```

**Allow monitoring (Prometheus):**

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        name: monitoring
  ports:
  - protocol: TCP
    port: 8080  # metrics port

```

### Cleanup

```bash
kubectl delete networkpolicy --all
kubectl delete pod web client

```

---
