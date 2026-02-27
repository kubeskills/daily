## Day 79: Supply Chain Attacks - Poisoned Dependencies

### Email Copy

**Subject:** Day 79: Supply Chain Attacks - When Your Base Image is Malicious

THE IDEA:
Pull container images and discover they're compromised. Backdoors in base images,
malicious dependencies, typosquatting attacks, and images with crypto miners.

THE SETUP:
Use untrusted registries, pull unverified images, deploy without scanning, skip
signature verification, and discover how supply chain attacks infiltrate clusters.

WHAT I LEARNED:
- Base images can contain backdoors
- npm/pip packages can be malicious
- Typosquatting: nginix vs nginx
- Image tags are mutable (latest changes)
- No signature verification = trust anyone
- Private registries can be compromised

WHY IT MATTERS:
Supply chain attacks cause:
- Backdoors in production
- Data exfiltration from containers
- Crypto mining on cluster resources
- Credentials stolen and leaked
- Complete cluster compromise

TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-079

Tomorrow: Rate limiting and API throttling failures.

---
Chad


---

## Killercoda Lab Instructions


## Step 1: Demonstrate image tag mutability

```bash
# Pull image with 'latest' tag
kubectl run test-latest --image=nginx:latest

# Tag is mutable - can change without notice
echo "Problem with 'latest' tag:"
echo "- Points to different image over time"
echo "- No guarantee of what you're running"
echo "- Can't reproduce deployments"
echo "- Security patches change behavior"
```

## Step 2: Test unverified image pull

```bash
# Pull from Docker Hub without verification
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: unverified-image
spec:
  containers:
  - name: app
    image: randomuser/suspicious-app:latest
    # No signature verification!
    # No vulnerability scanning!
    # Trust unknown publisher!
EOF

echo "Risks of unverified images:"
echo "- Unknown maintainer"
echo "- No security guarantees"
echo "- Could contain malware"
echo "- Could have backdoors"
```

<!-- Content was truncated. Please add the remaining steps. -->
