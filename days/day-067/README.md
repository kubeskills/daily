## Day 67: Init Containers - Startup Sequence Failures

### Email Copy

**Subject:** Day 67: Init Containers - When Startup Fails

THE IDEA:
Deploy pods with failing init containers and watch them get stuck in Init phase.
Main container never starts because prerequisites fail to complete.

THE SETUP:
Create init containers that crash, timeout, fail dependencies, and test what happens
when initialization doesn't complete successfully.

WHAT I LEARNED:
- Init containers run sequentially before main containers
- Pod stuck in Init:0/2 if first init container fails
- Init containers restart on failure (unless restartPolicy: Never)
- Sidecars (K8s 1.28+) run alongside main containers
- Init containers share volumes with main containers

WHY IT MATTERS:
Init container failures cause:
- Pods never become Ready (stuck in Init phase)
- Deployments stuck at 0/3 ready indefinitely
- Database migrations never run (app never starts)
- Difficult to debug (logs not obvious)

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-067

Tomorrow: Job and CronJob failures (Week 10 finale!).

---
Chad
Kube Daily - 67 days of production-ready failure knowledge


---

## Killercoda Lab Instructions

<!-- Lab instructions for this day coming soon. -->
