## Day 73: Multi-Tenancy Failures - Namespace Isolation Breaking

### Email Copy

**Subject:** Day 73: Multi-Tenancy Failures - When Tenants Access Each Other

THE IDEA:
Set up namespace isolation and watch it fail. Tenants access other namespaces, bypass
ResourceQuotas, escape NetworkPolicy restrictions, and view other teams' secrets.

THE SETUP:
Create tenant namespaces with quotas and policies, test cross-namespace access, break
RBAC boundaries, and discover what happens when isolation fails.

WHAT I LEARNED:
- Namespaces are NOT security boundaries alone
- RBAC required to prevent cross-namespace access
- NetworkPolicy blocks pod-to-pod, not API access
- ResourceQuota per namespace, but nodes shared
- PodSecurityPolicy applied cluster-wide
- Service accounts can access other namespaces with proper RBAC

WHY IT MATTERS:
Multi-tenancy failures cause:
- Tenant A reads Tenant B's secrets (data breach)
- Resource exhaustion affects all tenants
- Network policies bypassed via API server
- Noisy neighbor problem (one tenant impacts others)

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-073

Tomorrow: Backup and restore disasters.

---
Chad


---

## Killercoda Lab Instructions

<!-- Lab instructions for this day coming soon. -->
