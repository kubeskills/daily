## Day 40: Certificates - Expiration Catastrophe

## THE IDEA:
Discover a certificate is about to expire and watch API server connections fail,
webhooks break, and service mesh mTLS crumble. Learn to check cert validity.

## THE SETUP:
Check Kubernetes certificates, find expiration dates, simulate expired certs,
and test cert-manager failures and renewal problems.

## WHAT I LEARNED:
- Kubernetes certificates typically expire after 1 year
- Multiple certs: API server, kubelet, service account signing
- cert-manager automates renewal but can fail
- Expired certs cause immediate connection failures
- No grace period for expired certificates

## WHY IT MATTERS:
Certificate expiration causes:
- Complete cluster outages (API server unreachable)
- Webhook failures blocking all operations
- Service mesh traffic rejection

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-040

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/playgrounds/scenario/kubernetes


---

# Killercoda Lab Instructions


## Step 1: Check Kubernetes certificate locations
```bash
# List certificate files (if accessible)
ls -la /etc/kubernetes/pki/ 2>/dev/null || echo "PKI directory not accessible"

# Check certificate expiration
sudo kubeadm certs check-expiration 2>/dev/null || echo "kubeadm not available"
```

## Step 2: Check API server certificate
```bash
# Get API server endpoint
kubectl cluster-info | grep "Kubernetes control plane"

# Check cert validity (if accessible)
echo | openssl s_client -connect localhost:6443 2>/dev/null | openssl x509 -noout -dates
```

## Step 3: Install cert-manager
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Wait for pods
kubectl wait --for=condition=Ready pods --all -n cert-manager --timeout=120s
```

## Step 4: Create self-signed Issuer
```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
EOF
```

## Step 5: Create Certificate with short expiration
```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: short-lived-cert
spec:
  secretName: short-lived-tls
  duration: 24h  # 1 day
  renewBefore: 12h  # Renew when 12h left
  issuerRef:
    name: selfsigned-issuer
  dnsNames:
  - example.com
  - www.example.com
EOF
```

**Check certificate:**
```bash
kubectl get certificate short-lived-cert
kubectl describe certificate short-lived-cert
```

## Step 6: Check certificate secret
```bash
kubectl get secret short-lived-tls -o yaml

# Decode and check certificate
kubectl get secret short-lived-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text | grep -A 2 "Validity"
```

## Step 7: Simulate webhook with TLS
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: webhook-service
spec:
  selector:
    app: webhook
  ports:
  - port: 443
    targetPort: 8443
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webhook
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
        args: ["-text=webhook response", "-listen=:8443"]
        ports:
        - containerPort: 8443
        volumeMounts:
        - name: tls
          mountPath: /tls
          readOnly: true
      volumes:
      - name: tls
        secret:
          secretName: short-lived-tls
EOF
```

## Step 8: Create webhook certificate
```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: webhook-cert
spec:
  secretName: webhook-tls
  duration: 8760h  # 1 year
  renewBefore: 360h  # Renew 15 days before
  issuerRef:
    name: selfsigned-issuer
  dnsNames:
  - webhook-service.default.svc
  - webhook-service.default.svc.cluster.local
EOF
```

## Step 9: Test expired certificate simulation
```bash
# Create certificate that expires immediately
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: expired-cert
spec:
  secretName: expired-tls
  duration: 1h
  renewBefore: 59m  # Try to renew almost immediately
  issuerRef:
    name: selfsigned-issuer
  dnsNames:
  - expired.example.com
EOF
```

**Watch renewal:**
```bash
kubectl get certificate expired-cert -w
```

cert-manager continuously renews!

## Step 10: Test certificate renewal failure
```bash
# Delete issuer to break renewal
kubectl delete issuer selfsigned-issuer

# Wait for renewal attempt
sleep 30

# Check certificate status
kubectl describe certificate short-lived-cert | grep -A 10 Events
```

Shows renewal failures!

## Step 11: Restore issuer
```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
EOF

# Check certificate recovers
kubectl get certificate short-lived-cert -w
```

## Step 12: Test CA certificate
```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ca-cert
spec:
  isCA: true
  commonName: my-ca
  secretName: ca-secret
  duration: 43800h  # 5 years
  issuerRef:
    name: selfsigned-issuer
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: ca-issuer
spec:
  ca:
    secretName: ca-secret
EOF
```

**Issue cert from CA:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: app-cert
spec:
  secretName: app-tls
  duration: 2160h  # 90 days
  renewBefore: 360h  # Renew 15 days before
  issuerRef:
    name: ca-issuer
  dnsNames:
  - app.example.com
EOF
```

**Verify chain:**
```bash
kubectl get secret app-tls -o jsonpath='{.data.ca\.crt}' | base64 -d | openssl x509 -noout -subject
```

## Step 13: Test ClusterIssuer (cluster-wide)
```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: cluster-selfsigned
spec:
  selfSigned: {}
EOF
```

**Use in different namespace:**
```bash
kubectl create namespace other-ns

cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: cross-ns-cert
  namespace: other-ns
spec:
  secretName: cross-ns-tls
  issuerRef:
    name: cluster-selfsigned
    kind: ClusterIssuer
  dnsNames:
  - other.example.com
EOF
```

## Step 14: Check certificate readiness
```bash
kubectl get certificate -A
kubectl get certificaterequest -A
kubectl get order -A  # For ACME issuers
```

## Step 15: Debug certificate issues
```bash
# Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager --tail=50

# Describe certificate for events
kubectl describe certificate short-lived-cert

# Check secret contents
kubectl get secret short-lived-tls -o yaml

# Verify certificate
kubectl get secret short-lived-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text
```

## Key Observations

✅ **Kubernetes certs** - API server, kubelet, SA signing all expire
✅ **No grace period** - expired cert = immediate failure
✅ **cert-manager** - automates creation and renewal
✅ **renewBefore** - controls when renewal starts
✅ **CA chain** - must be valid for verification
✅ **ClusterIssuer** - works across all namespaces

## Production Patterns

**ACME/Let's Encrypt issuer:**
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

**Certificate for Ingress:**
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ingress-cert
spec:
  secretName: ingress-tls
  duration: 2160h  # 90 days
  renewBefore: 720h  # Renew 30 days before
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - example.com
  - www.example.com
```

**Monitor certificate expiration:**
```bash
# List all certificates with expiration
kubectl get certificate -A -o json | jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name): \(.status.notAfter)"'

# Alert on expiring certs (Prometheus)
# certmanager_certificate_expiration_timestamp_seconds - time() < 604800  # 7 days
```

**Backup certificates:**
```bash
# Export all certificate secrets
kubectl get secret -A -l cert-manager.io/certificate-name -o yaml > certs-backup.yaml
```

## Cleanup
```bash
kubectl delete certificate short-lived-cert expired-cert app-cert cross-ns-cert webhook-cert ca-cert 2>/dev/null
kubectl delete issuer selfsigned-issuer ca-issuer 2>/dev/null
kubectl delete clusterissuer cluster-selfsigned 2>/dev/null
kubectl delete deployment webhook 2>/dev/null
kubectl delete service webhook-service 2>/dev/null
kubectl delete namespace other-ns cert-manager 2>/dev/null
```

---
