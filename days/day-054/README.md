## Day 54: Certificate Rotation - Auto-Renewal Gone Wrong

### Email Copy

**Subject:** Day 54: Certificate Rotation - When Auto-Renewal Fails

THE IDEA:
Watch certificate auto-renewal fail and discover kubelet can't join the cluster,
webhooks stop working, and service mesh mTLS breaks. Learn what breaks when certs expire.

THE SETUP:
Check certificate expiration dates, simulate rotation failures, test cert-manager
renewal issues, and observe cascading failures from expired certificates.

WHAT I LEARNED:
- kubelet certificates auto-rotate (if enabled)
- CA certificate rotation requires manual intervention
- Webhook certificates must be updated in ValidatingWebhookConfiguration
- Service mesh certs managed separately (Istio, Linkerd)
- No warning before expiration (must monitor actively)

WHY IT MATTERS:
Certificate rotation failures cause:
- Nodes unable to join cluster (kubelet auth fails)
- All webhooks rejected (can't create/update resources)
- Service mesh traffic encrypted with expired certs (TLS errors)

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-054

🎉 Week 8 Complete! Disaster recovery and operations mastered!

---
Chad
Kube Daily - 54 days in, you're unstoppable

---

## Killercoda Lab Instructions

<!-- Lab instructions for this day coming soon. -->
