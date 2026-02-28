## Day 59: Webhook Timeouts - Admission Controller Chaos

### Email Copy

**Subject:** Day 59: Webhook Timeouts - When Admission Controllers Block Everything

THE IDEA:
Deploy validating and mutating webhooks that timeout or reject requests. Watch all
pod creations hang, kubectl commands fail, and the cluster become unusable.

THE SETUP:
Create webhooks with slow/unreachable endpoints, misconfigured failure policies,
and certificate issues. Test what happens when admission controllers fail.

WHAT I LEARNED:
- Webhooks can block ALL resource creation if misconfigured
- failurePolicy: Fail = block on webhook error (dangerous!)
- failurePolicy: Ignore = allow on error (safer)
- timeoutSeconds defaults to 10s (can cause slow operations)
- Webhook certificate issues block all matching operations
- namespaceSelector can exclude critical namespaces

WHY IT MATTERS:
Webhook failures cause:
- Complete cluster freeze (can't create any resources)
- kubectl commands timing out
- Unable to fix the issue (can't delete the broken webhook!)

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-059

Tomorrow: Custom Controller crashes and reconciliation loops.

---
Chad



---

## Killercoda Lab Instructions

<!-- Lab instructions for this day coming soon. -->
