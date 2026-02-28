## Day 45: kubectl debug - Ephemeral Container Debugging

## THE IDEA:
Try to debug a distroless container with no shell using kubectl exec (fails),
then use kubectl debug with ephemeral containers to attach debugging tools.

## THE SETUP:
Deploy minimal containers (distroless, scratch-based), attempt normal debugging,
fail, then use ephemeral debugging containers and node debugging modes.

## WHAT I LEARNED:
- Distroless/scratch containers have no shell (exec fails)
- Ephemeral containers attach to running pod temporarily
- --target shares process namespace with target container
- Can debug nodes directly with privileged containers
- Debug containers removed on pod deletion

## WHY IT MATTERS:
Limited debugging capabilities cause:
- Inability to troubleshoot production issues
- Need to rebuild containers with debug tools
- Delayed incident resolution

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-045

Tomorrow: Readiness vs liveness probe tuning.


---

# Killercoda Lab Instructions


## Step 1: Deploy minimal container (no shell)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: minimal-app
spec:
  containers:
  - name: app
    image: gcr.io/distroless/static-debian11
    command: ['sleep', '3600']
EOF
```

## Step 2: Try normal exec (fails!)

```bash
kubectl exec minimal-app -- sh
```

**Error:** "OCI runtime exec failed: exec failed: unable to start container process: exec: \"sh\": executable file not found"

No shell available!

## Step 3: Use kubectl debug with ephemeral container

```bash
kubectl debug minimal-app -it --image=busybox --target=app
```

Attaches busybox container that shares process namespace!

**Check processes:**
```bash
ps aux
# Shows processes from both containers
```

Type `exit` to leave debug session.

## Step 4: Debug with different image

```bash
# Use full-featured debug image
kubectl debug minimal-app -it --image=nicolaka/netshoot --target=app
```

Now you have network debugging tools!

```bash
# Inside debug container
ip addr
netstat -tulpn
curl localhost:8080 2>/dev/null || echo "No service on 8080"
```

## Step 5: Check ephemeral containers in pod

```bash
kubectl get pod minimal-app -o jsonpath='{.spec.ephemeralContainers}' | jq .
```

Shows attached debug containers!

## Step 6: Debug by copying pod

```bash
kubectl debug minimal-app -it --copy-to=minimal-app-debug --image=busybox
```

Creates copy of pod with busybox added as regular container.

**Check:**
```bash
kubectl get pods | grep minimal-app
kubectl describe pod minimal-app-debug
```

## Step 7: Debug with modified command

```bash
kubectl debug minimal-app -it --copy-to=minimal-debug-2 --container=app --image=busybox -- sh
```

Replaces container image and command!

## Step 8: Debug node directly

```bash
# Get node name
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')

# Debug node with privileged container
kubectl debug node/$NODE -it --image=ubuntu
```

Now you're in a privileged container on the node!

```bash
# Inside node debug container
chroot /host  # Access node's filesystem

# Check node processes
ps aux | head -20

# Check kubelet logs
tail /var/log/syslog | grep kubelet

# Exit chroot
exit
# Exit container
exit
```

## Step 9: Debug CrashLoopBackOff pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: crashloop
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'echo "Starting..."; exit 1']
EOF
```

**Pod crashes immediately:**
```bash
kubectl get pod crashloop
```

**Debug it:**
```bash
kubectl debug crashloop -it --copy-to=crashloop-debug --container=app --image=busybox -- sh
```

Override failing command to investigate!

## Step 10: Debug with profile

```bash
# Use debugging profile
kubectl debug minimal-app -it --profile=general --image=ubuntu
```

Profiles: general, baseline, restricted, netadmin, sysadmin

## Step 11: Debug with custom environment

```bash
kubectl debug minimal-app -it --image=busybox --target=app --env="DEBUG=true" --env="VERBOSE=1"
```

## Step 12: Check process tree from debug container

```bash
kubectl debug minimal-app -it --image=busybox --target=app

# Inside debug container
ps auxf  # Tree view
cat /proc/1/cmdline  # Check init process
ls -la /proc/1/root/  # View target's root filesystem
```

## Step 13: Debug with network tools

```bash
kubectl debug minimal-app -it --image=nicolaka/netshoot --target=app

# Inside debug container
# DNS debugging
nslookup kubernetes.default
dig kubernetes.default.svc.cluster.local

# Network connectivity
ping -c 3 8.8.8.8
traceroute google.com

# Port scanning
nc -zv localhost 1-1000

# HTTP debugging
curl -v http://kubernetes.default.svc
```

## Step 14: Debug filesystem issues

```bash
kubectl debug minimal-app -it --image=busybox --target=app

# Inside debug container
df -h  # Disk usage
mount  # Mounted filesystems
ls -la /proc/1/root/  # Target container's filesystem
```

## Step 15: Clean up ephemeral containers

```bash
# Ephemeral containers can't be removed directly
# They're removed when pod is deleted
kubectl delete pod minimal-app

# Clean up debug copies
kubectl delete pod minimal-app-debug minimal-debug-2 crashloop-debug
```

## Key Observations

✅ **Ephemeral containers** - attach to running pod temporarily
✅ **--target** - shares process namespace with target container
✅ **--copy-to** - creates pod copy with debug container
✅ **Node debugging** - debug node with privileged container
✅ **No removal** - ephemeral containers removed on pod deletion
✅ **Profiles** - preset security contexts for common scenarios

## Production Patterns

**Debug distroless container:**
```bash
kubectl debug <pod> -it \
  --image=busybox \
  --target=<container> \
  -- sh
```

**Debug networking:**
```bash
kubectl debug <pod> -it \
  --image=nicolaka/netshoot \
  --target=<container>
```

**Debug with full toolkit:**
```bash
kubectl debug <pod> -it \
  --image=ubuntu \
  --target=<container> \
  -- bash
```

**Debug node:**
```bash
kubectl debug node/<node-name> -it \
  --image=ubuntu
```

**Debug and override command:**
```bash
kubectl debug <pod> -it \
  --copy-to=<pod>-debug \
  --container=<container> \
  --image=busybox \
  -- sh
```

**Security-conscious debugging:**
```bash
kubectl debug <pod> -it \
  --profile=restricted \
  --image=busybox \
  --target=<container>
```

## Cleanup

```bash
kubectl delete pod minimal-app crashloop
```

---
