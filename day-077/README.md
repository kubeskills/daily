## Day 77: Secret Management - Credentials Everywhere

### Email Copy

**Subject:** Day 77: Secret Management - When Secrets Aren't Secret

THE IDEA:
Store secrets in Kubernetes and watch them leak. Secrets in logs, environment variables
visible in /proc, base64 decoded by anyone, and credentials committed to Git.

THE SETUP:
Create secrets incorrectly, expose them in logs, commit to Git, mount in wrong places,
and discover how easily credentials leak in Kubernetes.

WHAT I LEARNED:
- Secrets are base64 encoded, NOT encrypted
- Anyone with kubectl access can decode secrets
- Secrets logged to stdout = visible everywhere
- Environment variables visible in /proc filesystem
- Secret mounted as volume = readable by all containers in pod
- RBAC required to protect secret access

WHY IT MATTERS:
Secret leaks cause:
- Database credentials stolen
- API keys exposed in logs
- Credentials in Git history forever
- Compliance violations (PCI, HIPAA, SOC2)

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-077

Tomorrow: Container escape and privilege escalation.

---
Chad


---

## Killercoda Lab Instructions


## Step 1: Create basic secret

```bash
# Create secret
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=supersecret123

# View secret (base64 encoded)
kubectl get secret db-creds -o yaml
```

## Step 2: Decode secret (trivially easy)

```bash
# Anyone with kubectl access can decode
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d
echo ""

echo "Secrets are base64 encoded, NOT encrypted!"
echo "base64 is encoding, not encryption"
```

## Step 3: Secret in environment variable

```bash
# Deploy pod with secret as env var
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: env-secret-pod
spec:
  containers:
  - name: app
    image: nginx
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: password
EOF

kubectl wait --for=condition=Ready pod env-secret-pod --timeout=60s

# Secret visible in /proc
kubectl exec env-secret-pod -- cat /proc/1/environ | tr '\0' '\n' | grep DB_PASSWORD
```

Secret exposed in process environment!

## Step 4: Secret logged to stdout

```bash
# Pod that logs secret
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: logging-secret
spec:
  containers:
  - name: app
    image: busybox
    command:
    - sh
    - -c
    - |
      echo "Starting application..."
      echo "Database password: \$DB_PASSWORD"
      echo "Connecting to database..."
      sleep 3600
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: password
EOF

kubectl wait --for=condition=Ready pod logging-secret --timeout=60s

# Secret in logs!
kubectl logs logging-secret | grep -i password
```

Secret leaked to logs - visible to anyone with log access!

## Step 5: Secret in Git repository

```bash
# BAD: Secret checked into Git
mkdir -p /tmp/git-leak
cat > /tmp/git-leak/secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: api-key
type: Opaque
data:
  key: c3VwZXJzZWNyZXRhcGlrZXkxMjM=  # base64("supersecretapikey123")
EOF

echo "Secret in Git:"
echo "- Visible in Git history forever"
echo "- Accessible to anyone with repo access"
echo "- Shows up in GitHub search"
echo "- Rotated secrets still in history"

# Decode the "secret" from Git
echo "Decoded from Git:"
echo "c3VwZXJzZWNyZXRhcGlrZXkxMjM=" | base64 -d
echo ""
```

## Step 6: Secret without RBAC protection

```bash
# Create service account
kubectl create serviceaccount app-sa

# Deploy pod with ServiceAccount
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-reader
spec:
  serviceAccountName: app-sa
  containers:
  - name: app
    image: bitnami/kubectl
    command: ['sleep', '3600']
EOF

kubectl wait --for=condition=Ready pod secret-reader --timeout=60s

# Try to read secrets (fails without RBAC)
kubectl exec secret-reader -- kubectl get secrets 2>&1 || echo "Access denied (good!)"

# Grant access (bad idea in production)
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader-role
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secret-reader-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: secret-reader-role
subjects:
- kind: ServiceAccount
  name: app-sa
EOF

# Now pod can read all secrets!
kubectl exec secret-reader -- kubectl get secrets
kubectl exec secret-reader -- kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d
```

## Step 7: Secret mounted as volume

```bash
# Mount secret as volume
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: volume-secret-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-creds
EOF

kubectl wait --for=condition=Ready pod volume-secret-pod --timeout=60s

# Secret files readable
kubectl exec volume-secret-pod -- ls -la /etc/secrets
kubectl exec volume-secret-pod -- cat /etc/secrets/password
```

## Step 8: Secret in ConfigMap (wrong!)

```bash
# Using ConfigMap for secret data
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: wrong-secret
data:
  api-key: "my-super-secret-key-12345"
  db-password: "admin123"
EOF

# ConfigMaps are NOT for secrets!
kubectl get configmap wrong-secret -o yaml

echo "ConfigMaps:"
echo "- Visible in plain text"
echo "- No access control by default"
echo "- Not designed for sensitive data"
```

## Step 9: Secret in image

```bash
# Dockerfile with hardcoded secret (bad!)
cat > /tmp/bad-dockerfile << 'EOF'
FROM nginx
ENV API_KEY=supersecretkey123
COPY config.yaml /etc/config.yaml
# config.yaml contains passwords!
EOF

echo "Secrets in images:"
echo "- Baked into image layers"
echo "- Visible with 'docker history'"
echo "- Pushed to registry"
echo "- Can't rotate without rebuilding"
echo "- Anyone who pulls image gets secrets"
```

## Step 10: Secret visible in pod spec

```bash
# Create pod with secret
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pod-spec-secret
spec:
  containers:
  - name: app
    image: nginx
    env:
    - name: PASSWORD
      value: "hardcoded-secret-123"  # BAD!
EOF

# Secret visible in pod spec
kubectl get pod pod-spec-secret -o yaml | grep -A 2 "PASSWORD"

echo "Hardcoded secrets in manifests:"
echo "- Visible in Git"
echo "- Visible in cluster"
echo "- Visible to anyone with kubectl access"
```

## Step 11: Test SealedSecret (proper way)

```bash
# Conceptual: SealedSecret workflow
cat > /tmp/sealed-secret-example.yaml << 'EOF'
# 1. Create secret locally
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  password: c3VwZXJzZWNyZXQ=

# 2. Encrypt with kubeseal (one-way)
# kubeseal < secret.yaml > sealed-secret.yaml

# 3. SealedSecret (safe to commit to Git)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: mysecret
spec:
  encryptedData:
    password: AgBX7VvO3D... (encrypted, safe in Git)

# 4. Controller decrypts in cluster only
# Creates Secret automatically
# Only cluster can decrypt
EOF

cat /tmp/sealed-secret-example.yaml

echo ""
echo "SealedSecret benefits:"
echo "- Encrypted at rest in Git"
echo "- Only cluster can decrypt"
echo "- Asymmetric encryption (one-way)"
```

## Step 12: Secret in error messages

```bash
# Application that exposes secrets in errors
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: error-secret-pod
spec:
  containers:
  - name: app
    image: busybox
    command:
    - sh
    - -c
    - |
      DB_PASS=\$DB_PASSWORD
      echo "Trying to connect to database..."
      # Simulate connection error
      echo "ERROR: Failed to connect to db with password: \$DB_PASS" >&2
      exit 1
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: password
EOF

sleep 10

# Secret in error output
kubectl logs error-secret-pod 2>&1 | grep ERROR
```

Secret leaked in error messages!

## Step 13: Secret rotation without restart

```bash
# Update secret
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=newsecret456 \
  --dry-run=client -o yaml | kubectl apply -f -

# Pods using env vars don't see update
kubectl exec env-secret-pod -- printenv | grep DB_PASSWORD

echo "Secret updated in cluster, but:"
echo "- Pods with env vars still have old value"
echo "- Must restart pod to get new secret"
echo "- Volume mounts update automatically (eventually)"
```

## Step 14: Secret in backup

```bash
# Backup namespace
kubectl get secrets -o yaml > /tmp/secrets-backup.yaml

# Backup contains all secrets in plain(base64)
cat /tmp/secrets-backup.yaml | grep "password:"

echo "Secrets in backups:"
echo "- Base64 encoded"
echo "- Easily decoded"
echo "- Must encrypt backup files"
echo "- Secure backup storage critical"
```

## Step 15: Proper secret management

```bash
cat > /tmp/secret-management-guide.md << 'EOF'
# Proper Secret Management in Kubernetes

## DO NOT

- Store secrets in Git (even base64 encoded)
- Log secrets to stdout/stderr
- Hardcode secrets in Dockerfiles
- Use ConfigMaps for sensitive data
- Grant broad secret access via RBAC
- Mount secrets as env vars if avoidable

## DO

- Use external secret managers (Vault, AWS Secrets Manager)
- Use Kubernetes tools (SealedSecrets, External Secrets Operator)
- Enable encryption at rest for etcd
- Implement least-privilege RBAC
- Rotate secrets regularly
- Audit secret access
- Scan for leaked secrets
- Mount secrets as volumes (not env vars)
EOF

cat /tmp/secret-management-guide.md
```

## Key Observations

✅ **base64 ≠ encryption** - anyone can decode
✅ **Env vars exposed** - visible in /proc filesystem
✅ **Logs leak secrets** - avoid logging credentials
✅ **Git is forever** - never commit secrets
✅ **RBAC required** - protect secret access
✅ **Volume mounts better** - than environment variables

## Production Patterns

**Encrypted secret at rest:**
```yaml
# /etc/kubernetes/encryption/config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}
```

**Proper secret consumption:**
```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      serviceAccountName: app-sa
      containers:
      - name: app
        image: myapp:v1.0
        volumeMounts:
        - name: db-creds
          mountPath: /etc/secrets/db
          readOnly: true
      volumes:
      - name: db-creds
        secret:
          secretName: db-creds
          defaultMode: 0400
```

**External Secrets Operator:**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: db-creds
    creationPolicy: Owner
  data:
  - secretKey: password
    remoteRef:
      key: db/config
      property: password
```

## Cleanup

```bash
kubectl delete pod env-secret-pod logging-secret secret-reader volume-secret-pod pod-spec-secret error-secret-pod 2>/dev/null
kubectl delete secret db-creds api-key 2>/dev/null
kubectl delete configmap wrong-secret 2>/dev/null
kubectl delete role secret-reader-role 2>/dev/null
kubectl delete rolebinding secret-reader-binding 2>/dev/null
kubectl delete serviceaccount app-sa 2>/dev/null
rm -rf /tmp/git-leak /tmp/*.yaml /tmp/*.md /tmp/bad-dockerfile /tmp/secrets-backup.yaml
```

---
Next: Day 78 - Container Escape and Privilege Escalation
