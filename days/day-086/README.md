## Day 86: CRD Validation Failures - When Custom Resources Won't Apply

### Email Copy

**Subject:** Day 86: CRD Validation Failures - When Custom Resources Won't Apply

THE IDEA:
Create Custom Resource Definitions and watch validation fail. Schema mismatches,
missing required fields, invalid enum values, and CRDs that break existing resources.

THE SETUP:
Deploy CRDs with broken schemas, create resources that violate validation, update
CRDs that break compatibility, and discover what fails with custom resources.

WHAT I LEARNED:
- CRDs define custom resource schemas
- OpenAPI v3 schema validates resources
- Required fields must be present
- Enum values restrict allowed options
- CRD updates can break existing resources
- Structural schemas required for defaulting

WHY IT MATTERS:
CRD failures cause:
- Custom resources rejected by API server
- Operators can't create resources
- Breaking changes on CRD updates
- Invalid resources persist in etcd
- No validation until apply time

TRY IT:
🧪 Interactive Lab: https://killercoda.com/playgrounds/scenario/kubernetes

Tomorrow: DaemonSet deployment failures across nodes.

---
Chad

---

## Killercoda Lab Instructions

<!-- Lab instructions for this day coming soon. -->
