## Day 80: Rate Limiting - API Throttling Gone Wrong

### Email Copy

**Subject:** Day 80: Rate Limiting - When the API Says "Too Many Requests"

THE IDEA:
Hit API rate limits and watch everything slow down. kubectl commands fail with 429,
controllers get throttled, webhooks time out, and cluster operations grind to a halt.

THE SETUP:
Spam the API server, deploy chatty controllers, create thousands of resources,
hit cloud provider limits, and discover what breaks when rate limits kick in.

WHAT I LEARNED:
- API server has default rate limits (400 QPS)
- Controllers can overwhelm API with list/watch
- kubectl commands share the same limits
- Cloud provider APIs have separate limits
- Priority and Fairness queues requests
- Throttling cascades through system

WHY IT MATTERS:
Rate limiting failures cause:
- kubectl commands timeout or fail
- Controllers can't reconcile resources
- Deployments stuck in progress
- Webhooks miss events
- Cluster appears frozen

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-080

Tomorrow: Chaos engineering and deliberate failures.

---
Chad

---

## Killercoda Lab Instructions

<!-- Lab instructions for this day coming soon. -->
