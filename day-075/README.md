## Day 75: Observability Failures - When Metrics Disappear

### Email Copy

**Subject:** Day 75: Observability Failures - When You Can't See Anything

THE IDEA:
Deploy monitoring stack and watch it fail. Metrics missing, logs not collected, traces
incomplete, and dashboards showing no data. Blind debugging in production.

THE SETUP:
Break Prometheus scraping, fill up storage, lose metrics history, drop logs, and
discover what happens when observability fails.

WHAT I LEARNED:
- Metrics scraped every 15s (default), storage limited
- Prometheus uses pull model (must reach targets)
- Logs need aggregation (lost when pod deleted)
- Cardinality explosion kills Prometheus
- Retention depends on storage size
- No metrics = no alerts = no notifications

WHY IT MATTERS:
Observability failures cause:
- Can't diagnose production issues (no metrics/logs)
- Alerts don't fire (monitoring broken)
- SLO violations undetected
- Extended MTTR (mean time to repair)

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-075

🎉 Week 11 Complete! Advanced operations mastered!

---
Chad


---

## Killercoda Lab Instructions

<!-- Lab instructions for this day coming soon. -->
