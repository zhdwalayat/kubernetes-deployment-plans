# K8 Planning Skill — Eval

This eval tests whether the K8 Planning Skill produces correct and complete
Kubernetes deployment plans across different application types and security levels.

---

## How to Run This Eval

For each test case below:
1. Provide the input to the K8 Planning Skill
2. Compare the output against the expected results checklist
3. Mark each criterion as pass or fail
4. A passing score is 80% or above per test case

---

## Test Case 1 — Simple 2-Component App (Low Security)

**Input:**
- App name: Blog Platform
- Components: Frontend, Backend API
- External access: Frontend only
- State: No persistent storage
- Security level: Low
- External integrations: None

**Expected Output Checklist:**

| Criterion | Expected | Pass/Fail |
|---|---|---|
| Namespace | 1 namespace created | |
| Deployment type | Both use Deployment (stateless) | |
| Frontend Service | LoadBalancer | |
| Backend Service | ClusterIP | |
| ConfigMap | Contains API URL and environment | |
| Secrets | At least 1 Secret with expiry annotation | |
| Secret rotation | Every 180 days (Low security) | |
| RBAC | 2 ServiceAccounts, 2 Roles, 2 RoleBindings | |
| HPA | Not required (no AI/worker components) | |
| NetworkPolicy | Not required (Low security) | |
| Communication diagram | Shows Frontend → Backend | |
| Summary table | Lists both components with resource limits | |

**Pass threshold: 9/12 criteria**

---

## Test Case 2 — Stateful App with Database (Medium Security)

**Input:**
- App name: Task Tracker
- Components: Web App, REST API, PostgreSQL Database
- External access: Web App only
- State: PostgreSQL needs persistent storage
- Security level: Medium
- External integrations: None

**Expected Output Checklist:**

| Criterion | Expected | Pass/Fail |
|---|---|---|
| Namespace | 1 namespace created | |
| Web App deployment | Deployment (stateless) | |
| REST API deployment | Deployment (stateless) | |
| PostgreSQL deployment | StatefulSet (stateful) | |
| Web App Service | LoadBalancer | |
| REST API Service | ClusterIP | |
| PostgreSQL Service | ClusterIP | |
| ConfigMap | Contains DB host, port, app config | |
| DB Secret | Separate Secret for DB credentials | |
| Secret rotation | Every 90 days (Medium security) | |
| RBAC | 3 ServiceAccounts, least privilege per component | |
| PostgreSQL RBAC | Most restricted — only reads its own Secret | |
| HPA | Not required (no AI/worker components) | |
| Communication diagram | Web App → REST API → PostgreSQL | |
| Summary table | All 3 components listed | |

**Pass threshold: 12/15 criteria**

---

## Test Case 3 — AI Worker App (Medium Security)

**Input:**
- App name: AI Content Generator
- Components: Dashboard UI, Content API, AI Worker
- External access: Dashboard UI only
- State: No persistent storage
- Security level: Medium
- External integrations: None

**Expected Output Checklist:**

| Criterion | Expected | Pass/Fail |
|---|---|---|
| Namespace | 1 namespace created | |
| All deployments | All use Deployment (stateless) | |
| Dashboard Service | LoadBalancer | |
| Content API Service | ClusterIP | |
| AI Worker Service | ClusterIP | |
| ConfigMap | Contains service URLs and config | |
| Secrets | At least 1 Secret with expiry annotation | |
| Secret rotation | Every 90 days (Medium security) | |
| RBAC | 3 ServiceAccounts, 3 Roles, 3 RoleBindings | |
| HPA on AI Worker | Yes — minReplicas 2, maxReplicas ≥ 4 | |
| HPA CPU threshold | Between 60–75% | |
| Communication diagram | UI → Content API → AI Worker | |
| Summary table | All 3 components with HPA noted | |

**Pass threshold: 10/13 criteria**

---

## Test Case 4 — High Security App with External Integrations

**Input:**
- App name: Financial Assistant
- Components: AI Agent, Payment Service, Notification Service
- External access: None — internal tool only
- State: No persistent storage
- Security level: High
- External integrations: Payment Service calls Stripe API, Notification calls SendGrid

**Expected Output Checklist:**

| Criterion | Expected | Pass/Fail |
|---|---|---|
| Namespace | 1 namespace with security label | |
| All deployments | All use Deployment (stateless) | |
| All Services | All ClusterIP (no external access) | |
| ConfigMap | No credentials — config only | |
| Stripe Secret | Separate Secret for Stripe API key | |
| SendGrid Secret | Separate Secret for SendGrid API key | |
| Agent Secret | Separate Secret for Agent credentials | |
| Secret rotation | Every 60 days (High security) | |
| Secret annotations | expiry-date and owner on every Secret | |
| RBAC resourceNames | Each pod restricted to its own Secret only | |
| NetworkPolicy | Defined for all 3 components | |
| NetworkPolicy ingress | Only AI Agent can initiate calls | |
| NetworkPolicy egress | Payment and Notification allow outbound | |
| HPA on AI Agent | Yes — handles variable workloads | |
| Communication diagram | Shows all internal + external connections | |
| Summary table | Security summary table included | |

**Pass threshold: 13/16 criteria**

---

## Test Case 5 — Edge Case: Single Component App

**Input:**
- App name: Static Portfolio Website
- Components: Frontend only
- External access: Yes
- State: No
- Security level: Low
- External integrations: None

**Expected Output Checklist:**

| Criterion | Expected | Pass/Fail |
|---|---|---|
| Namespace | 1 namespace created | |
| Deployment | Single Deployment | |
| Service | LoadBalancer | |
| ConfigMap | Minimal — environment config only | |
| Secret | At least 1 even if minimal | |
| RBAC | 1 ServiceAccount, 1 Role, 1 RoleBinding | |
| HPA | Not required | |
| NetworkPolicy | Not required (Low security) | |
| Summary table | Single row table | |

**Pass threshold: 7/9 criteria**

---

## Overall Skill Evaluation Summary

| Test Case | App Type | Security | Pass Threshold | Result |
|---|---|---|---|---|
| 1 | Simple 2-component | Low | 9/12 | |
| 2 | Stateful with DB | Medium | 12/15 | |
| 3 | AI Worker | Medium | 10/13 | |
| 4 | High security + integrations | High | 13/16 | |
| 5 | Single component | Low | 7/9 | |

**Skill passes overall eval if: 4 out of 5 test cases pass**

---

## Key Skill Behaviors Being Tested

| Behavior | Tested In |
|---|---|
| Chooses StatefulSet for stateful components | Test Case 2 |
| Chooses Deployment for stateless components | All test cases |
| Uses LoadBalancer only for public components | All test cases |
| Separates Secrets per external integration | Test Case 4 |
| Applies stricter rotation for High security | Test Case 4 |
| Adds HPA only for AI/worker components | Test Cases 3 and 4 |
| Adds NetworkPolicy only for High security | Test Case 4 |
| Uses resourceNames in RBAC for Secret isolation | Test Case 4 |
| Handles single component edge case cleanly | Test Case 5 |
