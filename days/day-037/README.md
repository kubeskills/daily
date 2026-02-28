## Day 37: Volume Mounts - Permission Denied Chaos

## THE IDEA:
Mount a volume with restrictive permissions and watch your app crash with
"permission denied" errors. Test fsGroup, runAsUser conflicts, and SELinux issues.

## THE SETUP:
Create pods with various volume types (emptyDir, hostPath, PVC) and trigger
permission errors by running as non-root, mismatched fsGroup, and readonly mounts.

## WHAT I LEARNED:
- fsGroup sets group ownership on volume mount
- runAsUser can conflict with volume permissions
- ReadOnly mounts prevent all writes
- SELinux can block even root access
- subPath has different permission behavior

## WHY IT MATTERS:
Volume permission issues cause:
- Apps crashing on startup unable to write logs
- Database pods failing to initialize
- Configuration files unreadable

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-037

Tomorrow: DNS resolution failures and CoreDNS issues.

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/playgrounds/scenario/kubernetes


---

# Killercoda Lab Instructions


## Step 1: Create pod with emptyDir (works by default)
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-test
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'echo "hello" > /data/file.txt; cat /data/file.txt; sleep 3600']
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    emptyDir: {}
EOF
```

**Check logs:**
```bash
kubectl logs emptydir-test
```

Works! emptyDir is writable by default.

## Step 2: Run as non-root, trigger permission error
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nonroot-fail
spec:
  securityContext:
    runAsUser: 1000
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'echo "hello" > /data/file.txt']
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    emptyDir: {}
EOF
```

**Check logs:**
```bash
kubectl logs nonroot-fail 2>&1
```

Still works! emptyDir allows writes for any user.

## Step 3: Test hostPath with permission error
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-denied
spec:
  securityContext:
    runAsUser: 1000
    runAsNonRoot: true
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'ls -la /host; echo "test" > /host/file.txt 2>&1 || echo "Permission denied!"']
    volumeMounts:
    - name: host
      mountPath: /host
  volumes:
  - name: host
    hostPath:
      path: /root  # Root-only directory
      type: Directory
EOF
```

**Check logs:**
```bash
kubectl logs hostpath-denied
```

Permission denied! User 1000 can't write to /root.

## Step 4: Use fsGroup to fix permissions
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: fsgroup-fix
spec:
  securityContext:
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c']
    args:
    - |
      echo "Checking ownership..."
      ls -la /data
      echo "Writing file..."
      echo "test" > /data/file.txt
      echo "Reading file..."
      cat /data/file.txt
      sleep 3600
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    emptyDir: {}
EOF
```

**Check ownership:**
```bash
kubectl exec fsgroup-fix -- ls -la /data
```

Shows group ownership = 2000 (fsGroup)

## Step 5: Test readOnly volume mount
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: readonly-cm
data:
  config.txt: "readonly content"
---
apiVersion: v1
kind: Pod
metadata:
  name: readonly-mount
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c']
    args:
    - |
      echo "Reading file..."
      cat /config/config.txt
      echo "Attempting write..."
      echo "new content" > /config/config.txt 2>&1 || echo "Write blocked (readonly)"
      sleep 3600
    volumeMounts:
    - name: config
      mountPath: /config
      readOnly: true
  volumes:
  - name: config
    configMap:
      name: readonly-cm
EOF
```

**Check logs:**
```bash
kubectl logs readonly-mount
```

Write blocked by readOnly flag!

## Step 6: Test subPath permission behavior
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: subpath-test
spec:
  securityContext:
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c']
    args:
    - |
      echo "Checking /data..."
      ls -la /data
      echo "Checking /data/subdir..."
      ls -la /data/subdir
      touch /data/test1.txt
      touch /data/subdir/test2.txt
      sleep 3600
    volumeMounts:
    - name: data
      mountPath: /data
    - name: data
      mountPath: /data/subdir
      subPath: subdir
  volumes:
  - name: data
    emptyDir: {}
EOF
```

**Check behavior:**
```bash
kubectl exec subpath-test -- ls -la /data
kubectl exec subpath-test -- ls -la /data/subdir
```

subPath may have different permissions!

## Step 7: Test PVC permission issues
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: pvc-permissions
spec:
  securityContext:
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c']
    args:
    - |
      echo "Volume permissions:"
      ls -la /data
      echo "Attempting write..."
      echo "test" > /data/file.txt
      cat /data/file.txt
      sleep 3600
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: test-pvc
EOF
```

**Check if it works:**
```bash
kubectl logs pvc-permissions
```

## Step 8: Test multiple containers sharing volume
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume
spec:
  securityContext:
    fsGroup: 3000
  containers:
  - name: writer
    image: busybox
    command: ['sh', '-c']
    args:
    - |
      while true; do
        echo "$(date) - writing" >> /shared/log.txt
        sleep 5
      done
    securityContext:
      runAsUser: 1000
    volumeMounts:
    - name: shared
      mountPath: /shared
  - name: reader
    image: busybox
    command: ['sh', '-c']
    args:
    - |
      while true; do
        echo "=== Latest log ==="
        tail -5 /shared/log.txt 2>&1 || echo "File not yet created"
        sleep 10
      done
    securityContext:
      runAsUser: 2000
    volumeMounts:
    - name: shared
      mountPath: /shared
  volumes:
  - name: shared
    emptyDir: {}
EOF
```

**Check both containers:**
```bash
kubectl logs shared-volume -c writer --tail=5
kubectl logs shared-volume -c reader --tail=10
```

Both can access due to fsGroup!

## Step 9: Test volume from Secret
```bash
kubectl create secret generic secret-test --from-literal=key=value

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume
spec:
  securityContext:
    runAsUser: 1000
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c']
    args:
    - |
      echo "Secret permissions:"
      ls -la /secrets
      echo "Reading secret:"
      cat /secrets/key
      echo "Attempting write:"
      echo "new" > /secrets/key 2>&1 || echo "Cannot write to secret volume"
      sleep 3600
    volumeMounts:
    - name: secret
      mountPath: /secrets
  volumes:
  - name: secret
    secret:
      secretName: secret-test
      defaultMode: 0400  # Read-only for owner
EOF
```

**Check:**
```bash
kubectl logs secret-volume
```

Secrets are readonly by default!

## Step 10: Test projected volume
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: projected-volume
spec:
  securityContext:
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'ls -la /projected; cat /projected/*; sleep 3600']
    volumeMounts:
    - name: all-in-one
      mountPath: /projected
  volumes:
  - name: all-in-one
    projected:
      defaultMode: 0444
      sources:
      - secret:
          name: secret-test
      - configMap:
          name: readonly-cm
      - downwardAPI:
          items:
          - path: "labels"
            fieldRef:
              fieldPath: metadata.labels
EOF
```

**Check combined volume:**
```bash
kubectl logs projected-volume
kubectl exec projected-volume -- ls -la /projected
```

## Step 11: Test initContainer volume preparation
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: init-prepares-volume
spec:
  initContainers:
  - name: setup
    image: busybox
    command: ['sh', '-c']
    args:
    - |
      echo "Setting up volume..."
      mkdir -p /data/app
      echo "config" > /data/app/config.txt
      chmod 777 /data/app
    volumeMounts:
    - name: data
      mountPath: /data
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'ls -la /data; cat /data/app/config.txt; sleep 3600']
    securityContext:
      runAsUser: 1000
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    emptyDir: {}
EOF
```

**Verify:**
```bash
kubectl logs init-prepares-volume -c setup
kubectl logs init-prepares-volume -c app
```

## Step 12: Test SecurityContext conflicts
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: security-conflict
spec:
  securityContext:
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'id; ls -la /data; sleep 3600']
    securityContext:
      runAsUser: 3000  # Overrides pod-level
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    emptyDir: {}
EOF
```

**Check effective user:**
```bash
kubectl logs security-conflict
```

Container-level securityContext wins!

## Step 13: Debug volume permissions
```bash
# Create debug pod with privileged access
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: volume-debug
spec:
  containers:
  - name: debug
    image: busybox
    command: ['sleep', '3600']
    securityContext:
      privileged: true
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    emptyDir: {}
EOF
```

**Check from inside:**
```bash
kubectl exec volume-debug -- sh -c 'ls -la /data; df -h /data; mount | grep /data'
```

## Key Observations

✅ **fsGroup** - sets group ownership on volumes
✅ **runAsUser** - must have write permission to volume
✅ **readOnly** - prevents all write operations
✅ **subPath** - may have different permission behavior
✅ **Secrets** - readonly by default (mode 0400)
✅ **Container-level** - securityContext overrides pod-level

## Production Patterns

**Standard non-root setup:**
```yaml
apiVersion: v1
kind: Pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
    volumeMounts:
    - name: data
      mountPath: /app/data
  volumes:
  - name: data
    emptyDir: {}
```

**Database with proper permissions:**
```yaml
apiVersion: v1
kind: Pod
spec:
  securityContext:
    runAsUser: 999  # postgres user
    fsGroup: 999
  containers:
  - name: postgres
    image: postgres:15
    volumeMounts:
    - name: pgdata
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: pgdata
    persistentVolumeClaim:
      claimName: postgres-pvc
```

**Init container for permissions:**
```yaml
initContainers:
- name: fix-permissions
  image: busybox
  command: ['sh', '-c', 'chown -R 1000:2000 /data && chmod -R 755 /data']
  securityContext:
    runAsUser: 0  # Run as root to change ownership
  volumeMounts:
  - name: data
    mountPath: /data
```

## Cleanup
```bash
kubectl delete pod emptydir-test nonroot-fail hostpath-denied fsgroup-fix readonly-mount subpath-test pvc-permissions shared-volume secret-volume projected-volume init-prepares-volume security-conflict volume-debug 2>/dev/null
kubectl delete pvc test-pvc 2>/dev/null
kubectl delete secret secret-test 2>/dev/null
kubectl delete configmap readonly-cm 2>/dev/null
```

---
