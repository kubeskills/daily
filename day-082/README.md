## Day 82: Configuration Drift - When Clusters Diverge

### Email Copy

**Subject:** Day 82: Configuration Drift - When Dev ≠ Staging ≠ Production

THE IDEA:
Deploy to multiple environments and watch them drift apart. Different versions in prod,
manual patches in staging, config changes not in Git, and no two clusters the same.

THE SETUP:
Apply changes manually, skip GitOps, patch without documenting, use kubectl edit, and
discover how clusters become snowflakes over time.

WHAT I LEARNED:
- Manual kubectl apply causes drift
- kubectl edit changes not tracked
- Secrets updated without Git commits
- ConfigMaps patched directly
- Different versions across environments
- No single source of truth

WHY IT MATTERS:
Configuration drift causes:
- Can't reproduce environments
- Staging doesn't match production
- Rollbacks don't work (unknown state)
- Debugging impossible (what's actually running?)
- Compliance violations (undocumented changes)

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-082

🎉 Week 12 Complete! DevOps & Security mastered!

---
Chad

---

## Killercoda Lab Instructions

<!-- Lab instructions for this day coming soon. -->
