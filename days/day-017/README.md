# Day 17: Admission Webhooks - The Deployment Blocker

## THE IDEA:
Deploy a ValidatingWebhookConfiguration that points to a non-existent service. 
Watch ALL deployments hang waiting for webhook timeout (30 seconds default).

## THE SETUP:
Create a webhook that validates pod creation, point it at a dead endpoint, and 
discover your entire cluster deployment process is now frozen.

## WHAT I LEARNED:
- Webhooks block API requests until they respond or timeout
- failurePolicy: Fail vs Ignore determines behavior on webhook failure
- namespaceSelector can exempt critical namespaces (like kube-system)
- Webhook timeout is 10s default (can be 1-30s)

## WHY IT MATTERS:
Broken webhooks cause:
- Cluster-wide deployment outages
- Unable to delete resources (finalizers stuck)
- Emergency fixes blocked by your own policy

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-017

Tomorrow: CronJob concurrency policies causing job pile-ups.

---

## Killercoda Lab Instructions

### Step 1: Create a broken ValidatingWebhookConfiguration

```bash
cat <<EOF | kubectl apply -f -
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: broken-webhook
webhooks:
- name: broken.example.com
  clientConfig:
    service:
      name: nonexistent-webhook
      namespace: default
      path: /validate
    caBundle: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURQekNDQWllZ0F3SUJBZ0lVRmFxK3E4L0N5dDRlSTZpRkpUdE1XTGxKRDhNd0RRWUpLb1pJaHZjTkFRRUwKQlFBd0x6RXRNQ3NHQTFVRUF3d2tRV1J0YVhOemFXOXVJRU52Ym5SeWIyeHNaWElnVjJWaWFHOXZheUJFWlcxdgpJRU5CTUI0WERUSXpNRFl5TmpFM01qQXdNRm9YRFRJek1EY3lOakUzTWpBd01Gb3dMekV0TUNzR0ExVUVBd3drClFXUnRhWE56YVc5dUlFTnZiblJ5YjJ4c1pYSWdWMlZpYUc5dmF5QkVaVzF2SUVOQk1JSUJJakFOQmdrcWhraUcKOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXRRcEhKdUZQT3NqalNFbjRUckNpdjBaNnZBRlpxV2c9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  rules:
  - operations: ["CREATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 10
  failurePolicy: Fail  # Blocks on failure!
EOF

```

### Step 2: Try to create a pod (hangs!)

```bash
# This will hang for 10 seconds then fail
time kubectl run test-pod --image=nginx

```

**Observe:**

- Takes exactly 10 seconds (timeoutSeconds)
- Error: "failed calling webhook '[broken.example.com](http://broken.example.com/)': Post ... connection refused"

ALL pod creations are now blocked!

### Step 3: Check webhook status

```bash
kubectl get validatingwebhookconfiguration broken-webhook -o yaml

```

### Step 4: Fix with failurePolicy: Ignore

```bash
cat <<EOF | kubectl apply -f -
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: broken-webhook
webhooks:
- name: broken.example.com
  clientConfig:
    service:
      name: nonexistent-webhook
      namespace: default
      path: /validate
    caBundle: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURQekNDQWllZ0F3SUJBZ0lVRmFxK3E4L0N5dDRlSTZpRkpUdE1XTGxKRDhNd0RRWUpLb1pJaHZjTkFRRUwKQlFBd0x6RXRNQ3NHQTFVRUF3d2tRV1J0YVhOemFXOXVJRU52Ym5SeWIyeHNaWElnVjJWaWFHOXZheUJFWlcxdgpJRU5CTUI0WERUSXpNRFl5TmpFM01qQXdNRm9YRFRJek1EY3lOakUzTWpBd01Gb3dMekV0TUNzR0ExVUVBd3drClFXUnRhWE56YVc5dUlFTnZiblJ5YjJ4c1pYSWdWMlZpYUc5dmF5QkVaVzF2SUVOQk1JSUJJakFOQmdrcWhraUcKOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXRRcEhKdUZQT3NqalNFbjRUckNpdjBaNnZBRlpxV2c9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  rules:
  - operations: ["CREATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 10
  failurePolicy: Ignore  # Changed!
EOF

```

**Try creating pod again:**

```bash
kubectl run test-pod2 --image=nginx

```

Succeeds immediately! Webhook failure is ignored.

### Step 5: Exempt critical namespaces

```bash
cat <<EOF | kubectl apply -f -
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: safe-webhook
webhooks:
- name: safe.example.com
  clientConfig:
    service:
      name: nonexistent-webhook
      namespace: default
      path: /validate
    caBundle: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURQekNDQWllZ0F3SUJBZ0lVRmFxK3E4L0N5dDRlSTZpRkpUdE1XTGxKRDhNd0RRWUpLb1pJaHZjTkFRRUwKQlFBd0x6RXRNQ3NHQTFVRUF3d2tRV1J0YVhOemFXOXVJRU52Ym5SeWIyeHNaWElnVjJWaWFHOXZheUJFWlcxdgpJRU5CTUI0WERUSXpNRFl5TmpFM01qQXdNRm9YRFRJek1EY3lOakUzTWpBd01Gb3dMekV0TUNzR0ExVUVBd3drClFXUnRhWE56YVc5dUlFTnZiblJ5YjJ4c1pYSWdWMlZpYUc5dmF5QkVaVzF2SUVOQk1JSUJJakFOQmdrcWhraUcKOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXRRcEhKdUZQT3NqalNFbjRUckNpdjBaNnZBRlpxV2c9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: ["apps"]
    apiVersions: ["v1"]
    resources: ["deployments"]
  namespaceSelector:
    matchExpressions:
    - key: environment
      operator: NotIn
      values: ["kube-system", "kube-public"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 5
  failurePolicy: Fail
EOF

```

**Test exemption:**

```bash
# Create namespace without label - webhook applies
kubectl create namespace test-app
kubectl create deployment nginx --image=nginx -n test-app
# Fails!

# Create namespace with exemption label
kubectl create namespace prod-app
kubectl label namespace prod-app environment=kube-system
kubectl create deployment nginx --image=nginx -n prod-app
# Succeeds!

```

### Step 6: Deploy a working webhook (simple example)

```bash
# Create webhook service first
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-webhook
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webhook
  template:
    metadata:
      labels:
        app: webhook
    spec:
      containers:
      - name: webhook
        image: hashicorp/http-echo
        args:
        - "-text={\\"apiVersion\\":\\"admission.k8s.io/v1\\",\\"kind\\":\\"AdmissionReview\\",\\"response\\":{\\"uid\\":\\"REQUEST_UID\\",\\"allowed\\":true}}"
        - "-listen=:8443"
        ports:
        - containerPort: 8443
---
apiVersion: v1
kind: Service
metadata:
  name: webhook-svc
spec:
  selector:
    app: webhook
  ports:
  - port: 443
    targetPort: 8443
EOF

```

Wait for webhook pod to be Running.

### Step 7: Create working webhook configuration

First, generate a self-signed cert (for demo):

```bash
# Generate cert
openssl req -x509 -newkey rsa:2048 -keyout /tmp/key.pem -out /tmp/cert.pem -days 365 -nodes -subj "/CN=webhook-svc.default.svc"

# Get CA bundle (base64 encoded cert)
CA_BUNDLE=$(cat /tmp/cert.pem | base64 | tr -d '\\n')

```

```bash
cat <<EOF | kubectl apply -f -
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: working-webhook
webhooks:
- name: working.example.com
  clientConfig:
    service:
      name: webhook-svc
      namespace: default
      path: /validate
    caBundle: $CA_BUNDLE
  rules:
  - operations: ["CREATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["configmaps"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 5
  failurePolicy: Ignore
EOF

```

**Note:** This is simplified - production webhooks need proper TLS.

### Step 8: Test object selector

```bash
cat <<EOF | kubectl apply -f -
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: labeled-webhook
webhooks:
- name: labeled.example.com
  clientConfig:
    service:
      name: webhook-svc
      namespace: default
      path: /validate
    caBundle: $CA_BUNDLE
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  objectSelector:
    matchLabels:
      validate: "true"  # Only pods with this label!
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 5
  failurePolicy: Ignore
EOF

```

### Step 9: MutatingWebhook example (conceptual)

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: pod-mutator
webhooks:
- name: mutate.example.com
  clientConfig:
    service:
      name: webhook-svc
      namespace: default
      path: /mutate
    caBundle: <BASE64_CA>
  rules:
  - operations: ["CREATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
  reinvocationPolicy: Never  # or IfNeeded
  timeoutSeconds: 5
  failurePolicy: Ignore

```

Mutating webhooks can modify objects before creation!

### Step 10: Debug webhook failures

```bash
# Check API server logs
kubectl logs -n kube-system -l component=kube-apiserver --tail=50

# Test webhook directly (if service is accessible)
kubectl run webhook-test --rm -it --restart=Never --image=curlimages/curl -- \\
  curl -k <https://webhook-svc.default.svc.443/validate>

```

### Key Observations

✅ **failurePolicy: Fail** - blocks everything on webhook failure
✅ **failurePolicy: Ignore** - continues on webhook failure (safer for non-critical)
✅ **namespaceSelector** - exempt critical namespaces
✅ **objectSelector** - filter by labels on objects
✅ **Timeout matters** - default 10s can seem like eternity during outage

### Production Patterns

**Safe webhook defaults:**

```yaml
webhooks:
- name: safe.example.com
  failurePolicy: Ignore  # Don't block cluster on webhook failure
  timeoutSeconds: 5      # Fast timeout
  namespaceSelector:
    matchExpressions:
    - key: admission-control
      operator: NotIn
      values: ["disabled"]  # Allow escape hatch
  sideEffects: None
  reinvocationPolicy: Never

```

**Critical security webhook:**

```yaml
webhooks:
- name: security.example.com
  failurePolicy: Fail    # Security is critical
  timeoutSeconds: 3      # Fast response required
  namespaceSelector:
    matchExpressions:
    - key: kubernetes.io/metadata.name
      operator: NotIn
      values: ["kube-system", "kube-public"]

```

**Development exemption pattern:**

```yaml
namespaceSelector:
  matchExpressions:
  - key: environment
    operator: In
    values: ["production", "staging"]
# Dev namespaces not selected, bypassing webhook

```

### Cleanup

```bash
kubectl delete validatingwebhookconfiguration broken-webhook safe-webhook working-webhook labeled-webhook 2>/dev/null
kubectl delete deployment simple-webhook 2>/dev/null
kubectl delete service webhook-svc 2>/dev/null
kubectl delete pod test-pod test-pod2 2>/dev/null
kubectl delete namespace test-app prod-app 2>/dev/null
rm -f /tmp/key.pem /tmp/cert.pem

```

---

Next: Day 18 - CronJob Concurrency Policy Chaos