## Day 87: DaemonSet Failures - When Node-Level Pods Won't Deploy

### Email Copy

**Subject:** Day 87: DaemonSet Failures - When Node-Level Pods Won't Deploy

THE IDEA:
Deploy DaemonSets and watch them fail to run on all nodes. Pods skip nodes due to
taints, node selectors mismatch, resource constraints, and tolerations missing.

THE SETUP:
Create DaemonSets with wrong selectors, test node taints blocking pods, exceed node
resources, and discover what fails when every node should run a pod but doesn't.

WHAT I LEARNED:
- DaemonSets run one pod per node
- Node selectors restrict which nodes
- Tolerations required for tainted nodes
- UpdateStrategy controls rolling updates
- Pods scheduled even without resources
- Can't scale DaemonSets (replicas ignored)

WHY IT MATTERS:
DaemonSet failures cause:
- Monitoring agents missing on nodes
- Log collection incomplete
- Network plugins not running everywhere
- Security agents have gaps
- Storage drivers unavailable

TRY IT:
🧪 Interactive Lab: https://killercoda.com/playgrounds/scenario/kubernetes

Tomorrow: Horizontal Pod Autoscaler edge cases and scaling failures.

---
Chad

---

## Killercoda Lab Instructions

<!-- Lab instructions for this day coming soon. -->
