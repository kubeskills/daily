# Day 28: Pod Security Admission - Policy Enforcement


## THE IDEA:
Enable Pod Security Admission on a namespace and watch privileged pods get
rejected at admission time. Learn the difference between privileged, baseline,
and restricted security profiles.

## THE SETUP:
Label a namespace with PSA enforcement, try to deploy a privileged container,
and discover the detailed error message listing every security violation. Then
fix the pod to comply.

## WHAT I LEARNED:
- PSA replaces deprecated PodSecurityPolicy (removed in K8s 1.25)
- Three profiles: privileged (unrestricted), baseline (minimal), restricted (hardened)
- Enforcement happens at admission time (before pod creation)
- Multiple violations listed in single error message

## WHY IT MATTERS:
Security policy failures cause:
- Deployments silently rejected with cryptic errors
- Development workflows broken by production security requirements
- Security vulnerabilities from misconfigured containers

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-028

🎉 Week 4 Complete! 28 days of Kubernetes failure mastery unlocked!

---

