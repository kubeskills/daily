## Day 72: Cluster Upgrades - Breaking Changes

### Email Copy

**Subject:** Day 72: Cluster Upgrades - When Upgrades Break Everything

THE IDEA:
Upgrade cluster components and watch API versions break, deprecated features removed,
and workloads fail after "successful" upgrade. Learn what breaks between versions.

THE SETUP:
Test deprecated API versions, remove PodSecurityPolicy (removed 1.25), break admission
webhooks, and discover incompatibilities between control plane and nodes.

WHAT I LEARNED:
- API versions deprecate with 3-release warning
- PodSecurityPolicy removed in 1.25 (use Pod Security Standards)
- Control plane can be N+1 ahead of nodes
- Admission webhooks must support new API versions
- CRDs may need updates before cluster upgrade
- kubectl version should match cluster (±1 version)

WHY IT MATTERS:
Upgrade failures cause:
- Workloads fail to deploy (old API versions removed)
- Security policies stop working (PSP removed)
- Webhooks reject all requests (version mismatch)
- Complete cluster outage if upgrade goes wrong

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-072

Tomorrow: Multi-tenancy failures and namespace isolation.

---
Chad


---

## Killercoda Lab Instructions


## Step 1: Check current cluster version

```bash
# Get cluster version
kubectl version --short

# Get server version details
kubectl version -o json | jq -r '.serverVersion'

# Check node versions
kubectl get nodes -o wide
```

## Step 2: Check for deprecated API usage

```bash
# List all API versions
kubectl api-versions | sort

# Check for beta APIs (often deprecated)
kubectl api-resources --verbs=list -o wide | grep -v "^NAME" | sort

# Check for deprecated APIs in use
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis | head -10
```

<!-- Content was truncated. Please add the remaining steps. -->
