# Kubernetes Deployment Plans

Production-ready Kubernetes deployment plans for two AI-native application
scenarios. Includes a reusable K8 Planning Skill for generating deployment
plans for any future project.

## Contents

| File | Description |
|---|---|
| [Plan 1](./plan1-ai-task-manager.md) | AI Native Task Manager — UI, Backend APIs, Todo Agent, Notification Service |
| [Plan 2](./plan2-ai-employee-openclaw.md) | AI Employee (OpenClaw) — Personal AI with security focus |
| [K8 Planning Skill](./k8-planning-skill.md) | Reusable skill to generate K8 plans for any project |

## What Each Plan Covers

- Namespace isolation
- Deployments with resource requests and limits
- Services (LoadBalancer, ClusterIP)
- ConfigMaps for non-sensitive configuration
- Secrets with expiry annotations and rotation schedule
- RBAC — ServiceAccounts, Roles, RoleBindings
- Inter-service communication diagram
- HorizontalPodAutoscaler for AI workloads
- NetworkPolicies (Plan 2 — high security scenario)

## How to Use the K8 Planning Skill

Open `k8-planning-skill.md` and follow the 4 steps:
1. Gather inputs about your application
2. Apply decision rules to choose the right K8 resources
3. Fill in the output template
4. Verify against the checklist before submitting
