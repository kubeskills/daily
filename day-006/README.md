# Day 6: Liveness Probes - The Restart Loop of Death

## THE IDEA:
Misconfigure a liveness probe with a 1-second timeout on a slow endpoint. Watch
the pod enter an endless restart loop even though the app is healthy.

## THE SETUP:
Deploy a web server that takes 5 seconds to respond to health checks. Configure
a liveness probe with 1-second timeout and observe the restart cascade.

## WHAT I LEARNED:
- Liveness probe failures trigger restarts (not just marking unhealthy)
- successThreshold for liveness is always 1 (by design)
- failureThreshold determines how many failures before restart
- Restart backoff can delay recovery: 10s, 20s, 40s… up to 5 minutes

## WHY IT MATTERS:
Bad liveness probes are the #1 cause of self-inflicted outages:

- Slow database queries during startup kill pods repeatedly
- High CPU load makes probes timeout, triggering more load
- Too-aggressive probes prevent apps from ever becoming healthy

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-006

Tomorrow: Init containers that fail and block your main container forever.