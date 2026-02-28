## Day 60: Custom Controllers - Reconciliation Loop Hell

### Email Copy

**Subject:** Day 60: Custom Controllers - When Reconciliation Never Stops

THE IDEA:
Deploy a custom controller with bugs in its reconciliation logic and watch it loop
infinitely, thrash resources, and overwhelm the API server with requests.

THE SETUP:
Create controllers with missing status updates, conflicting desired state, infinite
retry loops, and resource leak patterns. Test what breaks when controllers fail.

WHAT I LEARNED:
- Controllers must update status or they reconcile forever
- Conflicting controllers fight over same resource
- Missing error handling causes immediate retries (API server overload)
- Watches on wrong resources trigger unnecessary reconciliation
- Leader election prevents split-brain in multi-replica controllers

WHY IT MATTERS:
Controller bugs cause:
- API server overwhelmed by reconciliation requests
- Resources constantly modified (flapping state)
- Controllers OOMKilled from leaked memory
- Cluster instability from cascading failures

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-060

Tomorrow: Helm chart deployment failures (Week 9 finale!).

---
Chad
Kube Daily - 60 days of systematic failure analysis



---

## Killercoda Lab Instructions

<!-- Lab instructions for this day coming soon. -->
