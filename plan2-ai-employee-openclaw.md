# Plan 2: Kubernetes Deployment Plan — AI Employee (OpenClaw)

## Application Overview

| Component | Role | External Access |
|---|---|---|
| Personal AI Employee | Core AI agent — triggered via secure API gateway or webhook, not directly exposed | No direct external access |
| Google Workspace Integration | Connects AI to Google Docs, Gmail, Calendar | No — outbound HTTPS only |
| WhatsApp/Discord Integration | Messaging interface for user interaction | No — outbound HTTPS only |
| LLM Service | Language model backend for AI reasoning | No — internal only |

---

## 1. Namespace

All components run in a dedicated namespace for strict isolation and security.
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ai-employee
  labels:
    environment: production
    security-level: high
```

> **Why:** A dedicated namespace allows tight RBAC control, network isolation,
> and prevents any cross-namespace access from other applications.

---

## 2. Deployments

### Personal AI Employee (Core Agent)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-employee
  namespace: ai-employee  ##This deployment handles both WhatsApp and Discord
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ai-employee
  template:
    metadata:
      labels:
        app: ai-employee
    spec:
      serviceAccountName: ai-employee-sa
      securityContext:
        runAsNonRoot: true
        allowPrivilegeEscalation: false
      containers:
      - name: ai-employee
        image: ai-employee:latest
        ports:
        - containerPort: 8000
        envFrom:
        - configMapRef:
            name: ai-employee-config
        - secretRef:
            name: ai-employee-secrets
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 15
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
```
```

### Google Workspace Integration
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: google-workspace
  namespace: ai-employee
spec:
  replicas: 1
  selector:
    matchLabels:
      app: google-workspace
  template:
    metadata:
      labels:
        app: google-workspace
    spec:
      serviceAccountName: google-workspace-sa
      securityContext:
        runAsNonRoot: true
        allowPrivilegeEscalation: false
      containers:
      - name: google-workspace
        image: google-workspace-integration:latest
        ports:
        - containerPort: 8001
        envFrom:
        - configMapRef:
            name: ai-employee-config
        - secretRef:
            name: google-secrets
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        livenessProbe:
          httpGet:
            path: /health
            port: 8001
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8001
          initialDelaySeconds: 5
          periodSeconds: 5
```
```
# Handles both WhatsApp and Discord messaging integrations
# WHATSAPP_API_TOKEN and DISCORD_BOT_TOKEN stored in messaging-secrets
```
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: messaging-integration
  namespace: ai-employee
spec:
  replicas: 1
  selector:
    matchLabels:
      app: messaging-integration
  template:
    metadata:
      labels:
        app: messaging-integration
    spec:
      serviceAccountName: messaging-sa
      securityContext:
        runAsNonRoot: true
        allowPrivilegeEscalation: false
      containers:
      - name: messaging-integration
        image: messaging-integration:latest
        ports:
        - containerPort: 8002
        envFrom:
        - configMapRef:
            name: ai-employee-config
        - secretRef:
            name: messaging-secrets
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        livenessProbe:
          httpGet:
            path: /health
            port: 8002
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8002
          initialDelaySeconds: 5
          periodSeconds: 5
```

### LLM Service
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llm-service
  namespace: ai-employee
spec:
  replicas: 2
  selector:
    matchLabels:
      app: llm-service
  template:
    metadata:
      labels:
        app: llm-service
    spec:
      serviceAccountName: llm-sa
      securityContext:
        runAsNonRoot: true
        allowPrivilegeEscalation: false
      containers:
      - name: llm-service
        image: llm-service:latest
        ports:
        - containerPort: 9000
        envFrom:
        - configMapRef:
            name: ai-employee-config
        - secretRef:
            name: llm-secrets
        resources:
          requests:
            cpu: "1000m"
            memory: "4Gi"
          limits:
            cpu: "4000m"
            memory: "8Gi"
        livenessProbe:
          httpGet:
            path: /health
            port: 9000
          initialDelaySeconds: 20
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 9000
          initialDelaySeconds: 10
          periodSeconds: 5
```

---

## 3. Services

All components use `ClusterIP` — no direct external exposure.
The AI Employee is triggered via a secure API gateway or webhook only.

| Component | Service Type | Port | Reason |
|---|---|---|---|
| AI Employee | ClusterIP | 8000 | Internal only — triggered via API gateway |
| Google Workspace | ClusterIP | 8001 | Internal only — called by AI Employee |
| WhatsApp/Discord | ClusterIP | 8002 | Internal only — called by AI Employee |
| LLM Service | ClusterIP | 9000 | Internal only — called by AI Employee |
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ai-employee-svc
  namespace: ai-employee
spec:
  type: ClusterIP
  selector:
    app: ai-employee
  ports:
  - port: 8000
    targetPort: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: google-workspace-svc
  namespace: ai-employee
spec:
  type: ClusterIP
  selector:
    app: google-workspace
  ports:
  - port: 8001
    targetPort: 8001
---
apiVersion: v1
kind: Service
metadata:
  name: messaging-svc
  namespace: ai-employee
spec:
  type: ClusterIP
  selector:
    app: messaging-integration
  ports:
  - port: 8002
    targetPort: 8002
---
apiVersion: v1
kind: Service
metadata:
  name: llm-svc
  namespace: ai-employee
spec:
  type: ClusterIP
  selector:
    app: llm-service
  ports:
  - port: 9000
    targetPort: 9000
```

---

## 4. ConfigMaps

Non-sensitive configuration only. No credentials, tokens, or keys here.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ai-employee-config
  namespace: ai-employee
data:
  GOOGLE_WORKSPACE_URL: "http://google-workspace-svc:8001"
  MESSAGING_URL: "http://messaging-svc:8002"
  LLM_SERVICE_URL: "http://llm-svc:9000"
  LOG_LEVEL: "info"
  ENVIRONMENT: "production"
  MAX_CONCURRENT_TASKS: "10"
  REQUEST_TIMEOUT_SECONDS: "30"
  LLM_MODEL: "gpt-4"
  RATE_LIMIT_RPM: "60"
  CIRCUIT_BREAKER_ENABLED: "true"
  RETRY_MAX_ATTEMPTS: "3"
```

---

## 5. Secrets

Each external integration has its own isolated Secret. Separating Secrets
limits blast radius — if one key is compromised, others remain safe.
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ai-employee-secrets       
  namespace: ai-employee
  annotations:
    secret-expiry-date: "2025-12-31"
    rotation-schedule: "every-60-days"
    owner: "platform-team"
type: Opaque
data:
  INTERNAL_API_KEY: <base64-encoded-value>
  SESSION_SECRET: <base64-encoded-value>
---
apiVersion: v1
kind: Secret
metadata:
  name: google-secrets
  namespace: ai-employee
  annotations:
    secret-expiry-date: "2025-12-31"
    rotation-schedule: "every-60-days"
    owner: "platform-team"
type: Opaque
data:
  GOOGLE_CLIENT_ID: <base64-encoded-value>
  GOOGLE_CLIENT_SECRET: <base64-encoded-value>
  GOOGLE_REFRESH_TOKEN: <base64-encoded-value>
---
apiVersion: v1
kind: Secret
metadata:
  name: messaging-secrets    #Contains both tokens:
  namespace: ai-employee
  annotations:
    secret-expiry-date: "2025-12-31"
    rotation-schedule: "every-60-days"
    owner: "platform-team"
type: Opaque
data:
  WHATSAPP_API_TOKEN: <base64-encoded-value>        #Both WhatsApp and Discord credentials are stored here separately.
  DISCORD_BOT_TOKEN: <base64-encoded-value>
---
apiVersion: v1
kind: Secret
metadata:
  name: llm-secrets
  namespace: ai-employee
  annotations:
    secret-expiry-date: "2025-12-31"
    rotation-schedule: "every-60-days"
    owner: "platform-team"
type: Opaque
data:
  LLM_API_KEY: <base64-encoded-value>
  LLM_ORG_ID: <base64-encoded-value>
```

**Secret Management Rules:**
- Each external integration has its own isolated Secret
- Secrets rotate every 60 days — stricter due to external API exposure
- All Secrets annotated with expiry date and owner for auditability
- In production: use HashiCorp Vault with dynamic secret injection

**Additional Security Practices:**
- Secrets are injected as environment variables or mounted as volumes
- Secrets are never logged or exposed in application responses
- Access is strictly controlled via RBAC resourceNames per component

---

## 6. RBAC — Roles and RoleBindings

Strict least-privilege access. Each component can only access its own Secret.
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ai-employee-sa
  namespace: ai-employee
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: google-workspace-sa
  namespace: ai-employee
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: messaging-sa
  namespace: ai-employee
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: llm-sa
  namespace: ai-employee
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ai-employee-role
  namespace: ai-employee
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
  resourceNames: ["ai-employee-secrets"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ai-employee-rolebinding
  namespace: ai-employee
subjects:
- kind: ServiceAccount
  name: ai-employee-sa
  namespace: ai-employee
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: ai-employee-role
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: google-workspace-role
  namespace: ai-employee
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
  resourceNames: ["google-secrets"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: google-workspace-rolebinding
  namespace: ai-employee
subjects:
- kind: ServiceAccount
  name: google-workspace-sa
  namespace: ai-employee
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: google-workspace-role
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: messaging-role
  namespace: ai-employee
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
  resourceNames: ["messaging-secrets"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: messaging-rolebinding
  namespace: ai-employee
subjects:
- kind: ServiceAccount
  name: messaging-sa
  namespace: ai-employee
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: messaging-role
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: llm-role
  namespace: ai-employee
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
  resourceNames: ["llm-secrets"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: llm-rolebinding
  namespace: ai-employee
subjects:
- kind: ServiceAccount
  name: llm-sa
  namespace: ai-employee
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: llm-role
```

---

## 7. Network Policies — Zero-Trust Security

Network policies restrict which pods can communicate with which, and
limit outbound traffic to only required external APIs.
```yaml
# AI Employee — only authorized internal services can communicate with it
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ai-employee-policy
  namespace: ai-employee
spec:
  podSelector:
    matchLabels:
      app: ai-employee
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api-gateway
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: google-workspace
    - podSelector:
        matchLabels:
          app: messaging-integration
    - podSelector:
        matchLabels:
          app: llm-service
  policyTypes:
  - Ingress
  - Egress
---
# Google Workspace — only AI Employee can call it, outbound HTTPS only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: google-workspace-policy
  namespace: ai-employee
spec:
  podSelector:
    matchLabels:
      app: google-workspace
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: ai-employee
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 443
  policyTypes:
  - Ingress
  - Egress
---
# Messaging Integration — only AI Employee can call it, outbound HTTPS only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: messaging-policy
  namespace: ai-employee
spec:
  podSelector:
    matchLabels:
      app: messaging-integration
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: ai-employee
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 443
  policyTypes:
  - Ingress
  - Egress
---
# LLM Service — only AI Employee can call it, outbound HTTPS only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: llm-policy
  namespace: ai-employee
spec:
  podSelector:
    matchLabels:
      app: llm-service
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: ai-employee
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 443
  policyTypes:
  - Ingress
  - Egress
```

**Policy Rules:**

| From | To | Allowed | Reason |
|---|---|---|---|
| API Gateway | AI Employee | ✅ | Entry point |
| AI Employee | Google Workspace | ✅ | Internal call |
| AI Employee | Messaging Integration | ✅ | Internal call |
| AI Employee | LLM Service | ✅ | Internal call |
| Google Workspace | Google APIs (HTTPS:443) | ✅ | Outbound only |
| Messaging Integration | WhatsApp/Discord (HTTPS:443) | ✅ | Outbound only |
| LLM Service | LLM Provider (HTTPS:443) | ✅ | Outbound only |
| Any pod | Any pod (not listed above) | ❌ | Blocked |
| Any pod | Any outbound port except 443 | ❌ | Prevents data exfiltration |

> **Why restrict egress to port 443 only:**
> Only HTTPS outbound is allowed. This prevents data exfiltration through
> non-standard ports and ensures all external communication is encrypted.

---

## 8. Inter-Service Communication
```
API Gateway / Webhook (secure entry point)
     │
     ▼
AI Employee (ClusterIP :8000)
     │
     ├──► Google Workspace Svc (ClusterIP :8001)
     │         │
     │         └──► Google APIs (HTTPS :443 outbound)
     │
     ├──► Messaging Svc (ClusterIP :8002)
     │         │
     │         └──► WhatsApp / Discord APIs (HTTPS :443 outbound)
     │
     └──► LLM Service (ClusterIP :9000)
               │
               └──► LLM Provider API (HTTPS :443 outbound)
```

| From | To | Protocol | Port | Type |
|---|---|---|---|---|
| API Gateway | AI Employee | HTTPS | 8000 | Internal |
| AI Employee | Google Workspace | HTTP | 8001 | Internal |
| AI Employee | Messaging Integration | HTTP | 8002 | Internal |
| AI Employee | LLM Service | HTTP | 9000 | Internal |
| Google Workspace | Google APIs | HTTPS | 443 | External outbound |
| Messaging Integration | WhatsApp/Discord | HTTPS | 443 | External outbound |
| LLM Service | LLM Provider | HTTPS | 443 | External outbound |

> **Note:** Only authorized internal services can communicate with the
> AI Employee, enforced via NetworkPolicies.

---

## 9. HorizontalPodAutoscaler

Both the AI Employee and LLM Service handle variable workloads and
benefit from autoscaling.
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ai-employee-hpa
  namespace: ai-employee
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ai-employee
  minReplicas: 2
  maxReplicas: 6
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 65
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: llm-hpa
  namespace: ai-employee
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: llm-service
  minReplicas: 2
  maxReplicas: 8
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

---

## 10. Secret Expiry, Compromise Handling and Agent Access Control

### What Happens When a Secret Expires
```
Secret approaches expiry date
        │
        ▼
Platform team notified via monitoring alert
        │
        ▼
New key generated at provider (Google, Discord, LLM)
        │
        ▼
New Kubernetes Secret created with updated value
        │
        ▼
Affected pod restarted to pick up new Secret
        │
        ▼
Old Secret deleted — expiry annotation updated
```

**Key rules:**
- Secrets rotated before expiry — not after
- Each Secret rotated independently — no cascading restarts
- In production: HashiCorp Vault handles auto-rotation

### What Happens When a Secret is Compromised

| Step | Action | Who |
|---|---|---|
| 1 | Revoke key immediately at provider | Platform team |
| 2 | Delete Kubernetes Secret immediately | Platform team |
| 3 | Generate new key from provider | Platform team |
| 4 | Create new Kubernetes Secret | Platform team |
| 5 | Restart only affected pod | Platform team |
| 6 | Audit logs for exposure source | Security team |
| 7 | Review RBAC — check no other pod accessed it | Security team |

### Agent Access Control

**Level 1 — Kubernetes RBAC:**
- AI Employee reads only its own Secret via `resourceNames`
- Cannot access Google, Discord or LLM Secrets

**Level 2 — NetworkPolicy:**
- Only API Gateway can reach AI Employee
- AI Employee can only reach its three integration services

**Level 3 — Pod Security Context:**
- All pods run as non-root
- Privilege escalation is disabled on all containers
```yaml
# Emergency: suspend AI Employee access
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ai-employee-sa
  namespace: ai-employee
  annotations:
    access-status: "suspended"
    suspended-date: "2025-03-29"
    suspended-reason: "Security investigation"
```

---

## 11. Audit and Monitoring

Security maturity requires visibility into all access and activity.

| What is Monitored | How | Why |
|---|---|---|
| All API access logs | Kubernetes audit logs | Detect unauthorized access |
| Secret access events | K8s audit policy on secrets | Alert on unexpected reads |
| Pod-to-pod traffic | NetworkPolicy + flow logs | Detect policy violations |
| Failed liveness probes | Prometheus + Alertmanager | Detect service degradation |
| Unusual egress traffic | Network flow monitoring | Detect data exfiltration |
| RBAC violations | K8s audit logs | Detect privilege abuse |

**Alerts triggered on:**
- Secret accessed by unexpected ServiceAccount
- Pod restarted more than 3 times in 5 minutes
- Outbound traffic on non-443 ports
- Failed readiness probe for more than 60 seconds

---

## 12. Advanced Reliability — Rate Limiting, Circuit Breaker and Retry Logic

### Rate Limiting
The AI Employee enforces a rate limit of 60 requests per minute to prevent
abuse and protect the LLM Service from overload.
```yaml
RATE_LIMIT_RPM: "60"   # configured via ConfigMap
```

### Circuit Breaker
If the LLM Service fails or becomes unresponsive, the circuit breaker
trips and returns a fallback response instead of cascading failures
across the system.
```
LLM Service fails 3 times
        │
        ▼
Circuit breaker trips — OPEN state
        │
        ▼
AI Employee returns fallback response to user
        │
        ▼
After 30 seconds — circuit breaker tries again (HALF-OPEN)
        │
        ▼
If LLM recovers → circuit breaker CLOSES — normal operation resumes
```

### Retry Logic
All external API calls (Google, Discord, LLM Provider) use retry logic
with exponential backoff to handle transient failures.
```yaml
RETRY_MAX_ATTEMPTS: "3"   # configured via ConfigMap
```

- Attempt 1 — immediate
- Attempt 2 — after 1 second
- Attempt 3 — after 3 seconds
- After 3 failures — circuit breaker triggers

---

## Summary Table

| Component | Type | Replicas | Service Type | CPU Limit | Memory Limit |
|---|---|---|---|---|---|
| AI Employee | Deployment | 2–6 (HPA) | ClusterIP | 2000m | 4Gi |
| Google Workspace | Deployment | 1 | ClusterIP | 500m | 512Mi |
| WhatsApp/Discord | Deployment | 1 | ClusterIP | 500m | 512Mi |
| LLM Service | Deployment | 2–8 (HPA) | ClusterIP | 4000m | 8Gi |

## Security Summary

| Security Measure | Applied To | Details |
|---|---|---|
| Separate Secrets | All 4 components | Each has its own isolated Secret |
| RBAC resourceNames | All 4 components | Each pod reads only its own Secret |
| NetworkPolicy ingress | All 4 components | Only authorized callers allowed |
| NetworkPolicy egress | Integration pods | HTTPS port 443 only — no exfiltration |
| Pod Security Context | All containers | Non-root, no privilege escalation |
| Secret rotation | All Secrets | Every 60 days with expiry annotations |
| Liveness probes | All components | Auto-restart on crash |
| Readiness probes | All components | Traffic only to healthy pods |
| Rate limiting | AI Employee | 60 RPM — prevents abuse |
| Circuit breaker | LLM Service | Fallback on failure |
| Retry logic | All external calls | 3 attempts with backoff |
| Audit logging | All components | Full access and anomaly monitoring |
