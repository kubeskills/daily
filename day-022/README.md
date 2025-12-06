## Day 22: Image Pull Secrets - Authentication Failures

## THE IDEA:
Deploy a pod that references a private container image without pull secrets.
Watch it get stuck in ImagePullBackOff forever. Learn to diagnose registry
authentication and secret linking.

## THE SETUP:
Create deployments that pull from private registries, test different secret
types (docker-registry vs dockerconfigjson), and explore automatic secret
attachment via ServiceAccount.

## WHAT I LEARNED:
- ImagePullBackOff = authentication or registry unreachable
- imagePullSecrets must be in same namespace as pod
- ServiceAccount can automatically attach pull secrets
- Secret type [kubernetes.io/dockerconfigjson](http://kubernetes.io/dockerconfigjson) format is specific

## WHY IT MATTERS:
Image pull failures cause:
- Pods stuck in pending forever with cryptic errors
- Deployment rollouts that hang indefinitely
- Production outages from expired registry credentials

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-022

Tomorrow: CNI plugin failures breaking pod networking.

---

# Killercoda Lab Instructions


## Step 1: Deploy pod with private image (no credentials)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: private-image-fail
spec:
  containers:
  - name: app
    image: private-registry.example.com/myapp:v1.0
EOF

```

**Watch it fail:**

```bash
kubectl get pod private-image-fail -w

```

Status: ImagePullBackOff or ErrImagePull

## Step 2: Diagnose the failure

```bash
kubectl describe pod private-image-fail | grep -A 10 Events

```

Look for:

- "Failed to pull image"
- "rpc error: code = Unknown desc = failed to pull and unpack image"
- Could be: unauthorized, DNS failure, or network timeout

## Step 3: Check pull attempts timing

```bash
kubectl get events --field-selector involvedObject.name=private-image-fail --sort-by='.lastTimestamp'

```

Note the backoff timing: immediate → 10s → 20s → 40s → up to 5 minutes

## Step 4: Create docker-registry secret (wrong format)

```bash
kubectl create secret generic broken-pull-secret \\
  --from-literal=username=myuser \\
  --from-literal=password=mypassword

```

**Try to use it:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: wrong-secret-type
spec:
  imagePullSecrets:
  - name: broken-pull-secret
  containers:
  - name: app
    image: private-registry.example.com/myapp:v1.0
EOF

```

**Check failure:**

```bash
kubectl describe pod wrong-secret-type | grep -A 5 Events

```

Error: Secret type mismatch or invalid format

## Step 5: Create proper docker-registry secret

```bash
kubectl create secret docker-registry correct-pull-secret \\
  --docker-server=private-registry.example.com \\
  --docker-username=myuser \\
  --docker-password=mypassword \\
  --docker-email=user@example.com

```

**Inspect the secret:**

```bash
kubectl get secret correct-pull-secret -o yaml

```

Type: `kubernetes.io/dockerconfigjson`

**Decode to see format:**

```bash
kubectl get secret correct-pull-secret -o jsonpath='{.data.\\.dockerconfigjson}' | base64 -d | jq .

```

Shows Docker config format with auth token.

## Step 6: Test with real public registry requiring auth

```bash
# GitHub Container Registry example
# Note: This will fail without real credentials

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ghcr-test
spec:
  imagePullSecrets:
  - name: ghcr-secret
  containers:
  - name: app
    image: ghcr.io/private-org/private-repo:latest
EOF

```

## Step 7: Create secret from existing Docker config

```bash
# If you have ~/.docker/config.json locally
# kubectl create secret generic regcred \\
#   --from-file=.dockerconfigjson=$HOME/.docker/config.json \\
#   --type=kubernetes.io/dockerconfigjson

# Simulate with manual creation
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: manual-docker-secret
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: eyJhdXRocyI6eyJwcml2YXRlLXJlZ2lzdHJ5LmV4YW1wbGUuY29tIjp7InVzZXJuYW1lIjoidGVzdCIsInBhc3N3b3JkIjoidGVzdCIsImF1dGgiOiJkR1Z6ZERwMFpYTjAifX19
EOF

```

## Step 8: Attach secret to ServiceAccount

```bash
kubectl create serviceaccount image-puller

kubectl patch serviceaccount image-puller -p '{"imagePullSecrets": [{"name": "correct-pull-secret"}]}'

```

**Verify:**

```bash
kubectl get serviceaccount image-puller -o yaml | grep -A 3 imagePullSecrets

```

**Use ServiceAccount in pod:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: auto-pull-secret
spec:
  serviceAccountName: image-puller
  containers:
  - name: app
    image: private-registry.example.com/myapp:v1.0
EOF

```

Secret automatically attached via ServiceAccount!

## Step 9: Multiple registry secrets

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multi-registry
spec:
  imagePullSecrets:
  - name: dockerhub-secret
  - name: ghcr-secret
  - name: gcr-secret
  containers:
  - name: app1
    image: private-repo/image1:latest
  - name: app2
    image: ghcr.io/org/image2:latest
  - name: app3
    image: gcr.io/project/image3:latest
EOF

```

Kubernetes tries each secret until one works!

## Step 10: Cross-namespace secret access (not allowed)

```bash
kubectl create namespace team-a
kubectl create namespace team-b

kubectl create secret docker-registry team-a-secret \\
  -n team-a \\
  --docker-server=registry.example.com \\
  --docker-username=user \\
  --docker-password=pass

# Try to use team-a secret from team-b
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cross-namespace-fail
  namespace: team-b
spec:
  imagePullSecrets:
  - name: team-a-secret  # Wrong namespace!
  containers:
  - name: app
    image: registry.example.com/private:latest
EOF

```

**Check error:**

```bash
kubectl describe pod cross-namespace-fail -n team-b | grep -A 5 Events

```

Error: secret not found (secrets are namespace-scoped)

## Step 11: Debug with manual image pull

```bash
# Create debug pod with docker/crictl
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: image-debug
spec:
  containers:
  - name: tools
    image: docker:latest
    command: ['sleep', '3600']
    securityContext:
      privileged: true
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
EOF

```

**Test manual pull:**

```bash
kubectl exec image-debug -- docker pull private-registry.example.com/myapp:v1.0

```

See the actual registry error!

## Step 12: Check kubelet image pull logs

```bash
# On node with failing pod
# journalctl -u kubelet | grep -i "image pull"

# Or check containerd logs
# journalctl -u containerd | grep -i "pull"

# In Killercoda, check pod events
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | grep -i image

```

## Step 13: Test image pull policy impact

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pull-policy-test
spec:
  containers:
  - name: app
    image: nginx:latest
    imagePullPolicy: Always  # Always pull, even if cached
EOF

```

**Policies:**

- `Always` - pull every time
- `IfNotPresent` - use cache if available
- `Never` - only use local cache

## Step 14: Simulate registry downtime

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: registry-timeout
spec:
  containers:
  - name: app
    image: nonexistent-registry.example.com:12345/app:latest
EOF

```

**Check error:**

```bash
kubectl describe pod registry-timeout | grep -A 5 Events

```

Different error: connection timeout vs authentication failure

## Step 15: Test image digest vs tag

```bash
# Using digest (immutable)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: digest-image
spec:
  containers:
  - name: app
    image: nginx@sha256:4c0fdaa8b6341bfdeca5f18f7837462c80cff90527ee35ef185571e1c327beac
EOF

```

Digests are immutable - no pull confusion!

## Key Observations

✅ **ImagePullBackOff** - authentication or registry issue
✅ **Secret namespace** - must match pod namespace
✅ **ServiceAccount attachment** - automatic secret injection
✅ **Secret type matters** - [kubernetes.io/dockerconfigjson](http://kubernetes.io/dockerconfigjson) required
✅ **Multiple secrets** - Kubernetes tries each sequentially

## Production Patterns

**Create from Docker config:**

```bash
kubectl create secret docker-registry regcred \\
  --from-file=.dockerconfigjson=$HOME/.docker/config.json \\
  --type=kubernetes.io/dockerconfigjson

```

**ServiceAccount with multiple registries:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
imagePullSecrets:
- name: dockerhub-creds
- name: gcr-creds
- name: ecr-creds

```

**Deployment with inline secret:**

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      imagePullSecrets:
      - name: private-registry-secret
      containers:
      - name: app
        image: private.registry/app:v1.0
        imagePullPolicy: IfNotPresent

```

**Create secret for multiple registries:**

```bash
kubectl create secret docker-registry multi-reg \\
  --docker-server=registry1.example.com \\
  --docker-username=user1 \\
  --docker-password=pass1

# Add more registries by patching
kubectl patch secret multi-reg -p '{"data":{".dockerconfigjson":"<base64-encoded-config>"}}'

```

## Cleanup

```bash
kubectl delete pod private-image-fail wrong-secret-type ghcr-test auto-pull-secret multi-registry cross-namespace-fail image-debug pull-policy-test registry-timeout digest-image 2>/dev/null
kubectl delete secret broken-pull-secret correct-pull-secret manual-docker-secret ghcr-secret team-a-secret 2>/dev/null
kubectl delete serviceaccount image-puller 2>/dev/null
kubectl delete namespace team-a team-b 2>/dev/null

```

---

Next: Day 23 - CNI Plugin Failures and Pod Networking