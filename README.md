# Kubernetes Deployment Plans

Production-ready Kubernetes deployment plans for two AI-native application
scenarios. Includes a reusable K8 Planning Skill and Eval for generating
and validating deployment plans for any future project.

---

## Contents

| File | Description |
|---|---|
| [Plan 1](./plan1-ai-task-manager.md) | AI Native Task Manager — UI, Backend APIs, Todo Agent, Notification Service |
| [Plan 2](./plan2-ai-employee-openclaw.md) | AI Employee (OpenClaw) — Personal AI with advanced security focus |
| [K8 Planning Skill](./k8-planning-skill.md) | Reusable skill to generate K8 plans for any project |
| [Eval](./eval.md) | 5 test cases to validate the K8 Planning Skill |

---

## What Each Plan Covers

### Plan 1 — AI Native Task Manager
- Namespace isolation
- 4 Deployments with resource requests, limits and health probes
- Ingress Controller for production-grade external access with TLS
- ClusterIP Services for all internal components
- ConfigMaps for non-sensitive configuration
- Secrets with expiry annotations and 90-day rotation schedule
- RBAC — dedicated ServiceAccounts, Roles and RoleBindings per component
- Network Policies — zero-trust pod-to-pod communication
- HorizontalPodAutoscaler for Todo Agent AI workload
- Pod Security Context — non-root, no privilege escalation
- Failure Handling and Reliability section
- Inter-service communication diagram

### Plan 2 — AI Employee (OpenClaw)
Everything in Plan 1 plus:
- Separate Secrets per external integration (Google, Discord, LLM)
- Egress restricted to HTTPS port 443 only — prevents data exfiltration
- Secret expiry and compromise response workflow
- Agent access control and revocation procedure
- Audit and monitoring strategy
- Rate limiting, Circuit Breaker and Retry logic
- HPA on both AI Employee and LLM Service

---

## How to Use the K8 Planning Skill

Open `k8-planning-skill.md` and follow 4 steps:

1. **Gather Inputs** — app name, components, access, state, security level
2. **Apply Decision Rules** — choose correct K8 resources for each component
3. **Generate Output Plan** — fill the 15-section template
4. **Verify Checklist** — 19-point checklist before submitting any plan

---

## Key Concepts Demonstrated

| Concept | Where Applied |
|---|---|
| Ingress over LoadBalancer | Plan 1 — Section 4 |
| Zero-trust NetworkPolicies | Plan 1 — Section 9, Plan 2 — Section 7 |
| Separate Secrets per integration | Plan 2 — Section 5 |
| RBAC resourceNames isolation | Both plans — RBAC section |
| Liveness and Readiness probes | Both plans — all Deployments |
| Pod Security Context | Both plans — all containers |
| HPA for AI workloads | Plan 1 — Todo Agent, Plan 2 — AI Employee + LLM |
| Secret rotation and compromise handling | Plan 2 — Section 10 |
| Circuit Breaker and Retry logic | Plan 2 — Section 12 |
| Resource Quotas | K8 Planning Skill — Section 15 |

---

## Technologies and Concepts Used

- Kubernetes Deployments, StatefulSets, Services, Ingress
- ConfigMaps and Secrets management
- RBAC — Roles, RoleBindings, ServiceAccounts
- NetworkPolicies — ingress and egress control
- HorizontalPodAutoscaler
- Pod Security Context
- Liveness and Readiness probes
- TLS termination via Ingress
- HashiCorp Vault (referenced for production Secret management)
- Prometheus and Alertmanager (referenced for monitoring)
