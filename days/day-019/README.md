# Day 19: ServiceAccount Tokens - Projection Problems

## THE IDEA:
Disable automountServiceAccountToken and watch your app fail mysteriously when 
it tries to call the Kubernetes API. Then explore token projection and expiration.

## THE SETUP:
Deploy a pod that needs API access but has automounting disabled. Discover the 
error messages and learn about projected volumes vs legacy tokens.

## WHAT I LEARNED:
- Legacy tokens never expire (security risk!)
- Projected tokens have bounded lifetime (1 hour default)
- automountServiceAccountToken can be set on SA or Pod
- Token path is /var/run/secrets/kubernetes.io/serviceaccount/token

## WHY IT MATTERS:
Token issues cause:
- Apps unable to access Kubernetes API
- Security vulnerabilities from long-lived credentials
- Token expiration breaking long-running processes

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-019

Tomorrow: Node NotReady and pod eviction behavior.

---


## Killercoda Lab Instructions

### Step 1: Check default token mounting

```bash
kubectl run default-token --image=nginx --dry-run=client -o yaml | grep -A 5 serviceAccount

```

No explicit serviceAccount means "default" SA is used with automounting enabled.

**Deploy and check:**

```bash
kubectl run default-token --image=nginx
kubectl exec default-token -- ls /var/run/secrets/kubernetes.io/serviceaccount/

```

Shows: ca.crt, namespace, token (all auto-mounted)

### Step 2: Disable automounting at pod level

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-token-pod
spec:
  automountServiceAccountToken: false
  containers:
  - name: app
    image: bitnami/kubectl:latest
    command: ['sleep', '3600']
EOF

```

**Check token directory:**

```bash
kubectl exec no-token-pod -- ls /var/run/secrets/kubernetes.io/serviceaccount/ 2>&1

```

Error: directory doesn't exist!

**Try to use kubectl:**

```bash
kubectl exec no-token-pod -- kubectl get pods

```

Error: "Unable to connect... Unauthorized"

### Step 3: Disable at ServiceAccount level

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-auto-sa
automountServiceAccountToken: false
---
apiVersion: v1
kind: Pod
metadata:
  name: sa-no-token
spec:
  serviceAccountName: no-auto-sa
  containers:
  - name: app
    image: nginx
EOF

```

**Verify no token:**

```bash
kubectl exec sa-no-token -- ls /var/run/secrets/kubernetes.io/serviceaccount/ 2>&1

```

SA-level setting also blocks token mounting.

### Step 4: Override SA setting at pod level

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: override-mount
spec:
  serviceAccountName: no-auto-sa
  automountServiceAccountToken: true  # Override SA setting!
  containers:
  - name: app
    image: nginx
EOF

```

**Check:**

```bash
kubectl exec override-mount -- ls /var/run/secrets/kubernetes.io/serviceaccount/

```

Token IS mounted - pod-level override works!

### Step 5: Compare legacy vs projected tokens

**Check API server for projected volume feature:**

```bash
kubectl explain pod.spec.volumes.projected

```

**Create pod with projected token:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: projected-token
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: token
      mountPath: /var/run/secrets/tokens
      readOnly: true
  volumes:
  - name: token
    projected:
      sources:
      - serviceAccountToken:
          path: api-token
          expirationSeconds: 3600  # 1 hour
          audience: api
EOF

```

**Check token:**

```bash
kubectl exec projected-token -- cat /var/run/secrets/tokens/api-token

```

Decode to see expiration:

```bash
TOKEN=$(kubectl exec projected-token -- cat /var/run/secrets/tokens/api-token)
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | jq .

```

Shows `exp` field (expiration timestamp).

### Step 6: Test token rotation

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: rotating-token
spec:
  containers:
  - name: app
    image: busybox
    command:
    - sh
    - -c
    - while true; do echo "Token at $(date):"; cat /var/run/secrets/tokens/token | cut -d. -f2 | base64 -d 2>/dev/null; sleep 300; done
    volumeMounts:
    - name: token
      mountPath: /var/run/secrets/tokens
  volumes:
  - name: token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 600  # 10 minutes
EOF

```

**Watch logs over time:**

```bash
kubectl logs rotating-token -f

```

Kubelet rotates token before expiration!

### Step 7: Multiple audiences

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multi-audience
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: api-token
      mountPath: /var/run/secrets/api
    - name: vault-token
      mountPath: /var/run/secrets/vault
  volumes:
  - name: api-token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          audience: api.kubernetes
          expirationSeconds: 3600
  - name: vault-token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          audience: vault
          expirationSeconds: 1800
EOF

```

Different tokens for different systems!

### Step 8: Token request API (programmatic)

```bash
# Create SA for testing
kubectl create sa token-requester

# Request token via API
kubectl create token token-requester --duration=10m

```

Returns a token valid for 10 minutes!

**Use it:**

```bash
TOKEN=$(kubectl create token token-requester --duration=1h)
kubectl --token=$TOKEN get pods

```

### Step 9: Legacy token secret (deprecated)

```bash
# Check if SA has auto-created secret (older K8s versions)
kubectl get sa default -o yaml | grep secrets -A 5

# In K8s 1.24+, no auto-created secrets
# Must manually create if needed (not recommended)

```

### Step 10: Bound service account tokens

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: bound-token
spec:
  containers:
  - name: app
    image: nginx
  volumes:
  - name: kube-api-access
    projected:
      sources:
      - serviceAccountToken:
          expirationSeconds: 3600
          path: token
      - configMap:
          name: kube-root-ca.crt
          items:
          - key: ca.crt
            path: ca.crt
      - downwardAPI:
          items:
          - path: namespace
            fieldRef:
              fieldPath: metadata.namespace
EOF

```

**This is what K8s does by default since 1.21!**

### Step 11: Test API access with projected token

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: api-caller
spec:
  serviceAccountName: default
  containers:
  - name: app
    image: bitnami/kubectl:latest
    command:
    - sh
    - -c
    - |
      while true; do
        kubectl get pods
        sleep 30
      done
EOF

```

**Check it works:**

```bash
kubectl logs api-caller

```

Should succeed with default SA permissions.

### Step 12: Verify token expiration behavior

```bash
# Create short-lived token
SHORT_TOKEN=$(kubectl create token default --duration=60s)

# Use it immediately
kubectl --token=$SHORT_TOKEN get pods

# Wait 90 seconds
sleep 90

# Try again
kubectl --token=$SHORT_TOKEN get pods

```

Error: token expired!

### Key Observations

✅ **automountServiceAccountToken** - can disable at SA or Pod level
✅ **Pod overrides SA** - pod setting takes precedence
✅ **Projected tokens** - expire and rotate automatically
✅ **Legacy tokens** - never expire (security risk, deprecated)
✅ **Token request API** - create short-lived tokens on demand

### Production Patterns

**Disable for non-API pods:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: webapp-sa
automountServiceAccountToken: false  # Web frontend doesn't need K8s API

```

**Operator with projected token:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: operator
spec:
  serviceAccountName: operator-sa
  containers:
  - name: operator
    image: operator:latest
    volumeMounts:
    - name: token
      mountPath: /var/run/secrets/kubernetes.io/serviceaccount
  volumes:
  - name: token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 7200  # 2 hours
      - configMap:
          name: kube-root-ca.crt
          items:
          - key: ca.crt
            path: ca.crt
      - downwardAPI:
          items:
          - path: namespace
            fieldRef:
              fieldPath: metadata.namespace

```

**External service with audience:**

```yaml
volumes:
- name: vault-token
  projected:
    sources:
    - serviceAccountToken:
        path: token
        audience: vault.example.com
        expirationSeconds: 3600

```

### Cleanup

```bash
kubectl delete pod default-token no-token-pod sa-no-token override-mount projected-token rotating-token multi-audience bound-token api-caller 2>/dev/null
kubectl delete sa no-auto-sa token-requester 2>/dev/null

```

---

Next: Day 20 - Node NotReady and Pod Eviction