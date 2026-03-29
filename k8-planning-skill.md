# K8 Planning Skill

A reusable skill for generating production-ready Kubernetes deployment plans
for any application. Provide the required inputs and this skill produces a
complete, structured plan covering all essential Kubernetes concepts.

---

## When to Use This Skill

Use this skill whenever you need to plan a Kubernetes deployment for:
- A new application with multiple components or microservices
- An AI-native application with agent-based architecture
- A security-sensitive application with external API integrations
- Any project requiring structured infrastructure planning before implementation

---

## Step 1: Gather Inputs

Before generating a plan, collect the following information about the project:

| Input | Question to Ask | Example |
|---|---|---|
| App name | What is the application called? | AI Task Manager |
| Components | List all components/services | UI, Backend, Agent, Notification |
| External access | Which components need public internet access? | UI only |
| State | Does any component store persistent data? | No — all stateless |
| Security level | Low / Medium / High | High |
| External integrations | Does any component call external APIs? | Yes — Google, Discord |
| Team size | How many teams/roles need access? | 2 teams |

---

## Step 2: Apply Decision Rules

Use these rules to decide the right Kubernetes resource for each component.

### Deployment Type
| Condition | Use |
|---|---|
| Component is stateless (no persistent storage) | `Deployment` |
| Component needs persistent storage (database, file store) | `StatefulSet` |
| Component must run on every node (logging, monitoring) | `DaemonSet` |

### Service Type
| Condition | Use |
|---|---|
| Component needs public internet access | `LoadBalancer` |
| Component is internal only | `ClusterIP` |
| Component needs access from within the cluster via node port | `NodePort` |

### Secret vs ConfigMap
| Data Type | Use |
|---|---|
| API keys, passwords, tokens, certificates | `Secret` |
| URLs, feature flags, environment names, timeouts | `ConfigMap` |

### Secret Rotation Schedule
| Security Level | Rotation Frequency |
|---|---|
| Low | Every 180 days |
| Medium | Every 90 days |
| High | Every 60 days |

### HPA (Auto Scaling)
| Condition | Add HPA |
|---|---|
| Component handles AI/ML workloads | Yes |
| Component receives unpredictable traffic spikes | Yes |
| Component is a background worker | Yes |
| Component is a simple static frontend | No |

---

## Step 3: Output Template

Use this template to generate the full plan. Replace all `<placeholders>`.
```markdown
