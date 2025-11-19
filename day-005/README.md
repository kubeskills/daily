## 🧪 **Pod Security Admission Lab**

This lab will show you how to:

1. Enable PSA on a namespace
2. Attempt to deploy a non-compliant pod
3. Modify the pod to comply with the `restricted` profile

---

### 📁 **Requirments**

A cluster running Kubernetes v1.25 or above.

---

### 📁 **Folder structure**

```
manifests/
├── namespace.yaml
├── privileged-pod.yaml
└── restricted-pod.yaml
```

---

### ✅ **Step 1: Create a namespace with PSA labels**

PSA is enforced via labels on the namespace.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: psa-lab
  labels:
    pod-security.kubernetes.io/enforce: "restricted"
    pod-security.kubernetes.io/enforce-version: "latest"
```

Apply it:

```bash
kubectl apply -f namespace.yaml
```

---

### 🚫 **Step 2: Try deploying a non-compliant privileged Pod**

This pod will violate the `restricted` policy because it uses `privileged: true`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
  namespace: psa-lab
spec:
  containers:
    - name: ubuntu
      image: ubuntu@sha256:6015f66923d7afbc53558d7ccffd325d43b4e249f41a6e93eef074c9505d2233
      command: [ "sh", "-c", "sleep 1h" ]
      securityContext:
        privileged: true
```

Try applying it:

```bash
kubectl apply -f privileged-pod.yaml
```

🔒 You should see an error like this:

```
Error from server (Forbidden): error when creating "privileged-pod.yaml": pods "privileged-pod" is forbidden: violates PodSecurity "restricted:latest": privileged (container "ubuntu" must not set securityContext.privileged=true), allowPrivilegeEscalation != false (container "ubuntu" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "ubuntu" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "ubuntu" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "ubuntu" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

---

### ✅ **Step 3: Fix the pod to comply with `restricted`**

Update the pod to meet the requirements:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
  namespace: psa-lab
spec:
  containers:
    - name: ubuntu
      image: ubuntu@sha256:6015f66923d7afbc53558d7ccffd325d43b4e249f41a6e93eef074c9505d2233
      command: [ "sh", "-c", "sleep 1h" ]
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
        seccompProfile:
          type: RuntimeDefault
```

Apply it:

```bash
kubectl apply -f restricted-pod.yaml
```

✅ This time, it should deploy successfully.

```bash
pod/restricted-pod created
```

---

### ✅ **Step 4: Cleanup**

To cleanup the environment, run the following command:

```bash
kubectl apply -f namespace.yaml
```
