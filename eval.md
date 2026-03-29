# K8 Planning Skill

A reusable skill for generating production-ready Kubernetes deployment plans
for any application. Provide the required inputs and this skill produces a
complete, structured plan covering all essential Kubernetes concepts.

---

## When to Use This Skill

Use this skill when:
- Designing microservices-based applications
- Planning AI/agent-based systems
- Deploying secure, API-integrated platforms
- Preparing infrastructure before application development begins

---

## Step 1: Gather Inputs

Before generating a plan, collect the following information:

| Input | Question to Ask | Example |
|---|---|---|
| App Name | What is the application called? | AI Task Manager |
| Components | What services exist? | UI, Backend, Agent |
| External Access | Which components need public access? | UI only |
| State | Any persistent storage needed? | No |
| Security Level | Low / Medium / High | High |
| External APIs | Any third-party integrations? | Google, Discord |
| Workload Type | Predictable or variable load? | Variable |
| Team Access | How many teams or users need access? | 2 teams |

---

## Step 2: Decision Rules

### 2.1 Workload Type — Resource Selection

| Condition | Kubernetes Resource |
|---|---|
| Stateless service | Deployment |
| Stateful service (DB, storage) | StatefulSet |
| Node-wide service (logging, monitoring) | DaemonSet |
| Scheduled job | CronJob |

### 2.2 Service Exposure

| Condition | Service Type |
|---|---|
| Public access required | ClusterIP + Ingress (preferred) OR LoadBalancer (basic setup) |
| Internal only | ClusterIP |
| Debug or testing only | NodePort |

> **Rule:** Always prefer Ingress over multiple LoadBalancers.
> Ingress provides centralized routing, SSL termination, and is more
> cost efficient in production. Use LoadBalancer only for simple or
> basic cloud setups where Ingress is not available.

### 2.3 Config vs Secret

| Data Type | Resource |
|---|---|
| Credentials, tokens, API keys | Secret |
| URLs, configs, feature flags | ConfigMap |

### 2.4 Secret Rotation Policy

| Security Level | Rotation Frequency |
|---|---|
| Low | Every 180 days |
| Medium | Every 90 days |
| High | Every 60 days |

### 2.5 Scaling Strategy

| Condition | Action |
|---|---|
| AI or agent workload | Add HPA |
| Unpredictable traffic spikes | Add HPA |
| Critical service | Use minimum 2 replicas |
| Heavy compute workload | Increase resource limits |

### 2.6 Security Rules — Mandatory for All Plans

Always apply these regardless of security level:
- Separate ServiceAccount per component
- RBAC with least privilege principle
- Isolated Secrets — no shared credentials across components
- NetworkPolicies for service-to-service isolation
- Pod Security Context — runAsNonRoot, no privilege escalation
- Never deploy resources in the default namespace

### 2.7 Risk and Security Handling

For High security applications, always include:
- Secret rotation workflow
- Compromise response plan
- Audit logging strategy
- Access revocation procedure

---

## Step 3: Output Template

Use this structure to generate the full plan for any application.

### 1. Namespace
- Create a dedicated namespace — never use the default namespace
- Add labels for environment and security level
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <app-name>
  labels:
    environment: production
    security-level: <low|medium|high>
```

### 2. Deployments / StatefulSets

For each component define:
- Type — Deployment (stateless) or StatefulSet (stateful)
- Replicas — minimum 2 for critical services
- Resource requests and limits — always set both
- ConfigMap and Secret references
- ServiceAccount assignment
- Liveness and Readiness probes
- Pod Security Context
- imagePullPolicy: Always — prevents outdated or vulnerable cached images
```yaml
containers:
- name: <component>
  image: <image>:latest
  imagePullPolicy: Always
  resources:
    requests:
      cpu: "<value>"
      memory: "<value>"
    limits:
      cpu: "<value>"
      memory: "<value>"
  livenessProbe:
    httpGet:
      path: /health
      port: <port>
    initialDelaySeconds: 10
    periodSeconds: 10
  readinessProbe:
    httpGet:
      path: /ready
      port: <port>
    initialDelaySeconds: 5
    periodSeconds: 5
  securityContext:
    runAsNonRoot: true
    allowPrivilegeEscalation: false
```

### 3. Services

For each component:
- Define Service type — ClusterIP or ClusterIP + Ingress
- Assign correct ports
- Justify the choice

| Component | Service Type | Reason |
|---|---|---|
| Public component | ClusterIP + Ingress (preferred) OR LoadBalancer (basic setup) | Centralized or direct external access |
| Internal component | ClusterIP | Internal communication only |

### 4. Ingress — Production Access Layer

Use Ingress for any component requiring external access.
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <app>-ingress
  namespace: <namespace>
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  rules:
  - host: <app>.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: <service-name>
            port:
              number: 80
  tls:
  - hosts:
    - <app>.example.com
    secretName: <tls-secret>
```

### 5. ConfigMaps

Store all non-sensitive configuration:
- Service URLs
- Environment name
- Feature flags
- Timeout values
- Rate limit settings

### 6. Secrets

- One Secret per component or external integration
- Annotate with expiry date, rotation schedule, and owner
- Apply rotation schedule based on security level
- Never store credentials in ConfigMaps or source code
```yaml
metadata:
  annotations:
    secret-expiry-date: "<date>"
    rotation-schedule: "every-<n>-days"
    owner: "<team>"
```

**Additional Security Practices:**
- Inject Secrets as environment variables or mounted volumes
- Never log or expose Secret values in responses
- Control access strictly via RBAC resourceNames

### 7. RBAC — Roles and RoleBindings

For each component:
- Create a dedicated ServiceAccount
- Create a Role with minimum required permissions
- Use `resourceNames` to restrict Secret access to only that component
- Bind Role to ServiceAccount via RoleBinding
```yaml
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
  resourceNames: ["<component-secret-name>"]
```

### 8. Network Policies — Zero-Trust Security

Restrict all pod-to-pod communication to only what is necessary.
```yaml
# Allow only specific callers into a pod
ingress:
- from:
  - podSelector:
      matchLabels:
        app: <allowed-caller>

# Restrict outbound to HTTPS only for external integrations
egress:
- to:
  - ipBlock:
      cidr: 0.0.0.0/0
  ports:
  - protocol: TCP
    port: 443
```

> **Why restrict egress to port 443:**
> Only HTTPS outbound is allowed. This prevents data exfiltration and
> ensures all external communication is encrypted.

### 9. Inter-Service Communication

- Define which service calls which
- Draw an ASCII flow diagram
- Use internal Kubernetes DNS for all internal calls:
```
<service-name>.<namespace>.svc.cluster.local
```

- Add a table: From → To → Protocol → Port → Internal/External

### 10. Scaling and Performance

For AI, worker, or variable traffic components add HPA:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  minReplicas: 2
  maxReplicas: <n>
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 65
```

### 11. Reliability and Health Checks

| Feature | Purpose |
|---|---|
| Liveness probe | Restarts crashed containers automatically |
| Readiness probe | Sends traffic only to healthy pods |
| Multiple replicas | Eliminates single point of failure |
| HPA | Handles load spikes automatically |
| imagePullPolicy: Always | Ensures latest secure image is always used |
| Pod Security Context | Prevents privilege escalation |

### 12. Security and Risk Handling

For High security applications include all of these:

**Secret Expiry Workflow:**
```
Secret approaches expiry
    │
    ▼
Team notified via alert
    │
    ▼
New key generated at provider
    │
    ▼
New Kubernetes Secret created
    │
    ▼
Affected pod restarted
    │
    ▼
Old Secret deleted — annotation updated
```

**Compromise Response:**
1. Revoke key at provider immediately
2. Delete Kubernetes Secret
3. Generate new key
4. Create new Secret
5. Restart affected pod only
6. Audit logs for exposure source
7. Review RBAC for unauthorized access

**Access Revocation:**
- Delete ServiceAccount token to instantly cut off pod access
- Redeploy with new ServiceAccount after investigation

### 13. Audit and Monitoring

| What | How | Why |
|---|---|---|
| API access logs | Kubernetes audit logs | Detect unauthorized access |
| Secret access | Audit policy on secrets | Alert on unexpected reads |
| Pod restarts | Prometheus alerts | Detect instability |
| Egress traffic | Network flow logs | Detect exfiltration |

### 14. Advanced Reliability — Rate Limiting, Circuit Breaker and Retry

**Rate Limiting:** Prevent abuse of AI agents — configure via ConfigMap.

**Circuit Breaker:** If a downstream service fails, trip the circuit and
return a fallback response instead of cascading failures.
```
Service fails 3 times → Circuit OPEN → Fallback response
After 30s → Circuit HALF-OPEN → Retry
If recovered → Circuit CLOSED → Normal operation
```

**Retry Logic:** For external API failures use exponential backoff.
- Attempt 1 — immediate
- Attempt 2 — after 1 second
- Attempt 3 — after 3 seconds
- After 3 failures — circuit breaker triggers

### 15. Resource Quotas — Optional Advanced

Limit total CPU and memory usage per namespace to prevent any single
application from exhausting cluster resources. Critical in multi-tenant
clusters.
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: <app>-quota
  namespace: <namespace>
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "20"
```

**Why Resource Quotas matter:**
- Prevents resource exhaustion from one namespace affecting others
- Enforces cost control in cloud environments
- Encourages developers to set proper resource requests and limits

---

## Step 4: Example Usage

**Input:**
- App: Chat Application
- Components: UI, Backend, Database
- External Access: UI only
- State: Database needs persistent storage
- Security Level: Medium
- External integrations: None

**Output Decisions:**

| Component | Type | Service | Notes |
|---|---|---|---|
| UI | Deployment | ClusterIP + Ingress | Public via Ingress |
| Backend | Deployment | ClusterIP | Internal only, add HPA |
| Database | StatefulSet | ClusterIP | Persistent storage |

- Secrets → DB credentials in separate Secret, rotation every 90 days
- ConfigMaps → API URLs, environment config
- RBAC → Backend can read DB Secret, UI cannot
- NetworkPolicy → Only Backend can reach Database
- HPA → Backend (variable traffic)
- Liveness + Readiness probes → All components
- imagePullPolicy: Always → All components
- No resources in default namespace

---

## Common Mistakes to Avoid

| Mistake | Correct Approach |
|---|---|
| Credentials in ConfigMaps | Always use Secrets |
| One shared Secret for all components | One Secret per component |
| Default ServiceAccount for all pods | Dedicated ServiceAccount per component |
| Deploying in default namespace | Always create a dedicated namespace |
| No resource limits on AI pods | Always set CPU and memory limits |
| Single replica for critical services | Minimum 2 replicas always |
| No HPA on AI workloads | Add HPA for any variable load component |
| No NetworkPolicies | Always define pod-to-pod communication rules |
| LoadBalancer for every service | Use Ingress for centralized external access |
| No liveness or readiness probes | Always add health checks |
| No imagePullPolicy | Set imagePullPolicy: Always on all containers |
| No secret rotation plan | Define rotation schedule and expiry annotations |

---

## Checklist Before Submitting Any K8 Plan

- [ ] Dedicated namespace with environment labels
- [ ] No resources deployed in default namespace
- [ ] Correct resource type — Deployment vs StatefulSet
- [ ] Ingress used for external access — not LoadBalancer directly
- [ ] No credentials in ConfigMaps
- [ ] One Secret per component with expiry annotation
- [ ] Dedicated ServiceAccount per component
- [ ] RBAC resourceNames restrict Secret access per pod
- [ ] Resource requests and limits on all containers
- [ ] imagePullPolicy: Always on all containers
- [ ] Liveness and Readiness probes on all containers
- [ ] Pod Security Context — runAsNonRoot, no privilege escalation
- [ ] HPA on AI or variable workload components
- [ ] NetworkPolicies define all allowed communication paths
- [ ] Egress restricted to port 443 for external integrations
- [ ] Inter-service communication diagram included
- [ ] Secret rotation workflow documented
- [ ] Audit and monitoring strategy included for High security
- [ ] Resource Quotas defined for namespace-level resource control

---

## Key Advantages of This Skill

- Standardized Kubernetes planning across any project
- Reduces design errors through structured decision rules
- Enforces security best practices by default
- Reusable for both simple and complex systems
- Covers full production requirements — not just basics
- Scales from a 2-component app to a complex AI platform
