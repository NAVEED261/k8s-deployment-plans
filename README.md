# Kubernetes Deployment Plans

Two infrastructure plans for AI-native applications — no code, just planning. The kind of planning that actually matters before you write a single line of YAML.

## What this covers

Most K8s tutorials stop at "here's how to run a pod." This project goes further — who can read which secret, what happens when the AI agent tries to access the database directly, how you rotate a JWT key without taking down the whole system. That's what these plans are about.

**Scenario 1 — AI Task Manager** (`plan1.md`)
UI, backend APIs, a task processing agent, and a notification service. Four namespaces keep the layers separate. The agent gets read access to exactly the secrets it needs — nothing more. Notifications go out through a ClusterIP service the backend calls internally; nothing about that service is reachable from outside the cluster.

**Scenario 2 — AI Employee using OpenClaw** (`plan2.md`)
This one got complicated fast. A personal AI worker with access to emails, calendar, and a code execution sandbox is a real security problem. The plan uses gVisor to isolate code execution at the kernel level, default-deny network policies across every namespace, and HashiCorp Vault for secret rotation. Every action the agent takes gets logged — immutably — to a separate audit namespace.

**K8 Planning Skill** (`k8-planning-skill.md`)
A reusable skill for generating K8s deployment plans for any future project. Feed it a list of services, a security level, and expected traffic — it walks through namespace design, workload classification, resource sizing, and a security checklist. Three eval test cases included for Anthropic's Skill Creator.

## Files

| File | Description |
|---|---|
| `plan1.md` | AI Native Task Manager — full deployment plan |
| `plan2.md` | AI Employee (OpenClaw) — security-first plan |
| `k8-planning-skill.md` | Reusable planning skill with eval |

## What each plan includes

- Namespaces for isolation
- Deployments and StatefulSets (with reasoning for each choice)
- Services — ClusterIP, NodePort, and LoadBalancer
- CPU and memory sizing per component
- ConfigMaps for non-sensitive config
- Secrets with rotation schedules
- RBAC roles, service accounts, and RoleBindings
- Inter-service DNS and NetworkPolicy rules
- Architecture diagrams (text-based)

---

**Author:** [NAVEED261](https://github.com/NAVEED261)
