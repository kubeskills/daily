# Day 10: Secret Rotation - Hot Swapping Credentials

## THE IDEA:
Rotate a database password in a Secret and discover your app is still using the
old credentials. Test different mounting strategies and explore external secret
operators.

## THE SETUP:
Deploy an app that reads credentials from a Secret. Rotate the secret and observe
which mounting methods pick up changes automatically.

## WHAT I LEARNED:
- Secret volume mounts update like ConfigMaps (~60s delay)
- Environment variables from Secrets are immutable (same as ConfigMaps)
- Secret data is base64 encoded, not encrypted at rest by default
- External Secrets Operator can sync from Vault/AWS Secrets Manager

## WHY IT MATTERS:
Credential rotation failures cause:
- Expired credentials crashing apps
- Security incidents from long-lived secrets
- Compliance violations (PCI-DSS, SOC2 require rotation)

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-010


---


## Killercoda Lab Instructions

### Step 1: Create initial secrets

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: admin
  password: supersecret123
  connection-string: "postgresql://admin:supersecret123@db:5432/myapp"
EOF

```

**Verify secret created:**

```bash
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d
echo ""

```

### Step 2: Deploy app with secret as env vars AND volume

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-app
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'while true; do echo "=== ENV VAR ==="; echo "DB_PASS=$DB_PASSWORD"; echo "=== VOLUME ==="; cat /secrets/password; echo ""; sleep 10; done']
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    volumeMounts:
    - name: secret-volume
      mountPath: /secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
EOF

```

**Watch the output:**

```bash
kubectl logs secret-app -f

```

Both show "supersecret123". Keep watching in this terminal.

### Step 3: Rotate the password

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: admin
  password: newsecurepassword456
  connection-string: "postgresql://admin:newsecurepassword456@db:5432/myapp"
EOF

```

**Keep watching logs for ~90 seconds:**

Observations:

- ENV VAR: Still shows "supersecret123" (never changes!)
- VOLUME: Updates to "newsecurepassword456" after ~60s

### Step 4: Verify the discrepancy

```bash
# Environment variable (frozen)
kubectl exec secret-app -- sh -c 'echo $DB_PASSWORD'

# Volume mount (updated)
kubectl exec secret-app -- cat /secrets/password

```

Your app sees TWO different passwords depending on how it reads them!

### Step 5: Projected volume with explicit items

```bash
kubectl delete pod secret-app

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: projected-secret
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'while true; do echo "Password:"; cat /secrets/db-pass; echo ""; sleep 10; done']
    volumeMounts:
    - name: secret-volume
      mountPath: /secrets
  volumes:
  - name: secret-volume
    projected:
      sources:
      - secret:
          name: db-credentials
          items:
          - key: password
            path: db-pass
            mode: 0400  # Read-only for owner
EOF

```

**Test rotation:**

```bash
kubectl logs projected-secret -f &

# Update secret
kubectl patch secret db-credentials -p '{"stringData":{"password":"rotated789"}}'

# Wait ~60s, then check
kubectl exec projected-secret -- cat /secrets/db-pass

```

### Step 6: Immutable secrets (prevent accidental changes)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: immutable-secret
type: Opaque
immutable: true
stringData:
  api-key: "fixed-key-never-changes"
EOF

```

**Try to update it:**

```bash
kubectl patch secret immutable-secret -p '{"stringData":{"api-key":"new-key"}}'

```

Error! Immutable secrets can't be modified. You must delete and recreate.

### Step 7: Secret with automatic rotation trigger

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rotating-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: rotating-app
  template:
    metadata:
      labels:
        app: rotating-app
      annotations:
        secret-version: "v1"
    spec:
      containers:
      - name: app
        image: busybox
        command: ['sh', '-c', 'echo "Starting with password: $DB_PASS"; sleep 3600']
        env:
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
EOF

```

**Rotate and trigger restart:**

```bash
# Update secret
kubectl patch secret db-credentials -p '{"stringData":{"password":"finalpassword000"}}'

# Trigger deployment rollout
kubectl patch deployment rotating-app -p '{"spec":{"template":{"metadata":{"annotations":{"secret-version":"v2"}}}}}'

# Watch rollout
kubectl rollout status deployment rotating-app

# Verify new password
kubectl logs -l app=rotating-app

```

### Step 8: Check secret encryption at rest

```bash
# View secret in etcd format (base64, not encrypted by default!)
kubectl get secret db-credentials -o yaml

```

The data is only base64 encoded. For actual encryption at rest, you need:

- EncryptionConfiguration on API server
- KMS provider (AWS KMS, GCP KMS, Vault)

### Step 9: Using hashicorp vault agent (conceptual)

```yaml
# This is a conceptual example - requires Vault installation
apiVersion: v1
kind: Pod
metadata:
  name: vault-app
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "myapp"
    vault.hashicorp.com/agent-inject-secret-db-creds: "secret/data/myapp/db"
    vault.hashicorp.com/agent-inject-template-db-creds: |
      {{- with secret "secret/data/myapp/db" -}}
      export DB_USER="{{ .Data.data.username }}"
      export DB_PASS="{{ .Data.data.password }}"
      {{- end }}
spec:
  containers:
  - name: app
    image: myapp:latest
    command: ['sh', '-c', 'source /vault/secrets/db-creds && ./start.sh']

```

Key Observations

✅ **Environment variables** - frozen at pod creation
✅ **Volume mounts** - update after kubelet sync (~60s)
✅ **Immutable secrets** - prevent accidental modification
✅ **base64 ≠ encryption** - secrets are not encrypted by default

Production Patterns

**External Secrets Operator:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials
  data:
  - secretKey: password
    remoteRef:
      key: prod/myapp/db
      property: password

```

**Sealed Secrets (GitOps-safe):**

```bash
# Encrypt secret for Git storage
kubeseal --format yaml < secret.yaml > sealed-secret.yaml

```

Cleanup

```bash
kubectl delete deployment rotating-app 2>/dev/null
kubectl delete pod secret-app projected-secret 2>/dev/null
kubectl delete secret db-credentials immutable-secret 2>/dev/null

```

---

Next: Day 11 - Service DNS Propagation Delays