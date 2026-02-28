## Day 88: HPA Edge Cases - When Autoscaling Makes Things Worse

### Email Copy

**Subject:** Day 88: HPA Edge Cases - When Autoscaling Makes Things Worse

THE IDEA:
Configure HPA and watch it make bad scaling decisions. Flapping between replicas,
scaling based on wrong metrics, thrashing during traffic spikes, and CPU metrics
that don't reflect actual load.

THE SETUP:
Create HPA with too-aggressive targets, test custom metrics failures, simulate
metric lag, and discover edge cases where autoscaling causes more problems.

WHAT I LEARNED:
- HPA uses 15s metric collection interval
- Scaling happens after 3min of metric threshold
- Scale-down has 5min stabilization window
- Custom metrics require metric server
- CPU/memory utilization = requests (not limits)
- Flapping prevented by cooldown periods

WHY IT MATTERS:
HPA edge cases cause:
- Constant pod churn (scaling up and down)
- Resource waste from over-scaling
- Performance degradation during scale-up
- OOMKills from under-scaling
- Unexpected costs from scale-out

TRY IT:
🧪 Interactive Lab: https://killercoda.com/playgrounds/scenario/kubernetes

Tomorrow: Network Policy failures and traffic blocking (Week 13 finale!).

---
Chad
Kube Daily - 88 days of production expertise!

---

## Killercoda Lab Instructions

<!-- Lab instructions for this day coming soon. -->
