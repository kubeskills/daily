# daily

<https://daily.kubeskills.com>

## 🧠 One K8s a Day

**Learn Kubernetes by testing one idea every day.**
Short, focused experiments that you can run on any cluster in under 10 minutes.
Each day explores a single Kubernetes concept — from Deployments and NetworkPolicies to Operators, Helm, and Kustomize.

---

## 🎯 Purpose

The goal is to **learn by doing** — and sometimes by breaking things.

Every experiment includes:
- **A clear objective** (what you'll learn)
- **A minimal YAML or CLI test**
- **A real outcome** (success or failure)
- **A short reflection** (what it means)

You'll develop muscle memory for daily Kubernetes operations and uncover how different parts of the system interact in practice.

---

## 🗂 Repository Structure

    daily/
    ├── day-001/
    │   ├── README.md        # Context, commands, and output
    │   └── manifests/       # YAML files
    ├── day-002/
    │   ├── README.md
    │   └── manifests/
    ├── day-003/
    │   └── …
    └── …

Each directory stands alone — no dependencies between days.
You can start anywhere, but following in order helps build context.

[Get started today with Day 1](day-001/README.md)


---

## 🚀 How to Use

### Option 1: Run in the browser with Killercoda (no system requirements)

Each day includes lightweight YAMLs safe to run in public sandboxes.


### Option 2: Run Locally with Kind or K3s

    # Create a local cluster
    kind create cluster --name k8s-lab

    # Apply today's manifests
    kubectl apply -f day-001/manifests/

    # Explore results
    kubectl get all -A

---


## 🧩 Topics Covered

| Area | Examples |
|------|-----------|
| Core Concepts | Pods, Deployments, ReplicaSets, Services |
| Configuration | ConfigMaps, Secrets, Volumes, Probes |
| Scheduling | NodeSelectors, Affinity, Taints, Tolerations |
| Security | RBAC, NetworkPolicies, Kyverno, Falco |
| GitOps | ArgoCD, FluxCD, Helm, Kustomize |
| Operators | CRDs, Controllers, Custom APIs |
| Networking | CoreDNS, Ingress, Cilium, Service Mesh |
| Storage | PVCs, StatefulSets, CSI, Rook, Longhorn |
| Observability | Prometheus, Grafana, OpenTelemetry, Logs |
| Troubleshooting | kubectl tips, Metrics, Events, Debug pods |
| Scaling & Reliability | HPA, KEDA, Cluster Autoscaler |
| Policy & Governance | PSPs (legacy), Pod Security, Admission Control |
| Advanced | Multi-cluster setups, CAPI, Edge, GPU workloads |


---

## 💡 Example Day Format

    Day 5 – Multi-Env Testing with Kustomize

    Goal: Patch different deployment variants.
    Command: kubectl apply -k kustomization-examples/env1
    Observation: env1-my-app deployed with its own labels.
    Lesson: Overlays make environment management trivial.


---

## 🧠 Why Daily Practice?
- Builds intuition faster than passive reading
- Reinforces core commands through repetition
- Encourages "fail fast" learning habits
- Helps you see Kubernetes as a system, not a checklist

---

## 📬 Subscribe

Get new experiments delivered daily:
👉 [Join the newsletter](https://daily.kubeskills.com)

Or follow along on Twitter/X: @KubeSkills

---

## 🤝 Contribute

Want to add your own "day"?
1. Fork the repo
2. Create a new folder `dayXX`
3. Add:
   - `README.md` (context, commands, lessons)
   - `manifests/` (YAML or Helm values)
4. Open a pull request

Your test idea might become part of the next release of the newsletter.

---

## 🧾 License

MIT © 2025 KubeSkills

---

"Fail fast. Learn faster. Ship knowledge daily."

---
