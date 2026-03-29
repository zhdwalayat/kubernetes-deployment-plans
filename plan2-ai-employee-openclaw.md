# Plan 2: Kubernetes Deployment Plan — AI Employee (OpenClaw)

## Application Overview

| Component | Role | External Access |
|---|---|---|
| Personal AI Employee | Core AI agent processing user requests | No — internal only |
| Google Workspace Integration | Connects AI to Google Docs, Gmail, Calendar | No — outbound only |
| WhatsApp/Discord Integration | Messaging interface for user interaction | No — outbound only |
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
  namespace: ai-employee
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
```

### WhatsApp/Discord Integration
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
```

---

## 3. Services

| Component | Service Type | Port | Reason |
|---|---|---|---|
| AI Employee | ClusterIP | 8000 | Internal only — orchestrates all other services |
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
```

---

## 5. Secrets — Security Focus

This scenario has three external service integrations, each with its own Secret.
Separating Secrets limits the blast radius — if one key is compromised,
the others remain safe.

### AI Employee Core Secrets
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
```

### Google Workspace Secrets
```yaml
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
```

### WhatsApp/Discord Secrets
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: messaging-secrets
  namespace: ai-employee
  annotations:
    secret-expiry-date: "2025-12-31"
    rotation-schedule: "every-60-days"
    owner: "platform-team"
type: Opaque
data:
  WHATSAPP_API_TOKEN: <base64-encoded-value>
  DISCORD_BOT_TOKEN: <base64-encoded-value>
```

### LLM Service Secrets
```yaml
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

**Secret Security Rules:**
- Each external integration has its own isolated Secret
- Secrets rotate every 60 days (stricter than Plan 1 due to external API exposure)
- All Secrets annotated with expiry date and owner for auditability
- In production: use HashiCorp Vault with dynamic secret injection
- Never log Secret values — ensure all services have log sanitization enabled

---

## 6. RBAC — Roles and RoleBindings

Strict least-privilege access. Each component can only access its own Secret.
```yaml
# ServiceAccounts
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
# AI Employee Role — reads config and its own secret only
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
# Google Workspace Role — reads its own secret only
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
# Messaging Role — reads its own secret only
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
# LLM Role — reads its own secret only
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

## 7. Network Policies

Since this is a security-focused scenario, NetworkPolicies restrict which pods
can talk to which — even within the same namespace.
```yaml
# Only AI Employee can call Google Workspace
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
  - to: []   # Allow outbound to Google APIs
---
# Only AI Employee can call Messaging Integration
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
  - to: []   # Allow outbound to WhatsApp/Discord APIs
---
# Only AI Employee can call LLM Service
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
  - to: []   # Allow outbound to LLM API provider
```

---

## 8. Inter-Service Communication
```
User Request
     │
     ▼
AI Employee (ClusterIP :8000)
     │
     ├──► Google Workspace Svc (ClusterIP :8001)
     │         │
     │         └──► Google APIs (external outbound)
     │
     ├──► Messaging Svc (ClusterIP :8002)
     │         │
     │         └──► WhatsApp / Discord APIs (external outbound)
     │
     └──► LLM Service (ClusterIP :9000)
               │
               └──► LLM Provider API (external outbound)
```

| From | To | Protocol | Port | Notes |
|---|---|---|---|---|
| AI Employee | Google Workspace | HTTP | 8001 | Internal |
| AI Employee | Messaging Integration | HTTP | 8002 | Internal |
| AI Employee | LLM Service | HTTP | 9000 | Internal |
| Google Workspace | Google APIs | HTTPS | 443 | External outbound |
| Messaging Integration | WhatsApp/Discord | HTTPS | 443 | External outbound |
| LLM Service | LLM Provider | HTTPS | 443 | External outbound |

---

## 9. HorizontalPodAutoscaler — AI Employee and LLM Service

Both the AI Employee and LLM Service handle variable workloads and benefit from autoscaling.
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

Secrets in this deployment are annotated with `secret-expiry-date` and
`rotation-schedule`. The following process applies when a Secret reaches
its expiry date:
```
Secret approaches expiry date
        │
        ▼
Platform team is notified (via monitoring alert or calendar reminder)
        │
        ▼
New Secret value is generated (new API key requested from provider)
        │
        ▼
New Secret is created in Kubernetes with updated value
        │
        ▼
Affected pod is restarted to pick up the new Secret
        │
        ▼
Old Secret is deleted
        │
        ▼
Expiry annotation is updated with new rotation date
```

**Key rules:**
- Secrets are rotated before expiry — not after
- Each Secret is rotated independently — rotating one does not affect others
- In production: use HashiCorp Vault with auto-rotation to eliminate manual steps

---

### What Happens When a Secret is Compromised

If a Secret is suspected to be leaked or compromised, the following
immediate response applies:

| Step | Action | Who |
|---|---|---|
| 1 | Revoke the compromised key immediately at the provider (Google, Discord, LLM) | Platform team |
| 2 | Delete the Kubernetes Secret immediately | Platform team |
| 3 | Generate a new key from the external provider | Platform team |
| 4 | Create a new Kubernetes Secret with the new value | Platform team |
| 5 | Restart only the affected pod | Platform team |
| 6 | Audit logs to identify how the Secret was exposed | Security team |
| 7 | Review RBAC to ensure no other pod accessed the Secret | Security team |

**Why separate Secrets per integration matter here:**
If all credentials were in one shared Secret, a single compromise would
take down all integrations. With separate Secrets, only the compromised
integration is affected — Google Workspace, WhatsApp/Discord, and LLM
Service remain fully operational independently.

---

### Agent Access Control

The Personal AI Employee pod runs under `ai-employee-sa` ServiceAccount
with strictly limited permissions. Access is controlled at three levels:

**Level 1 — Kubernetes RBAC:**
- The AI Employee can only read its own Secret (`ai-employee-secrets`)
- It cannot access Google, Discord or LLM Secrets
- It cannot create, update or delete any Kubernetes resources

**Level 2 — NetworkPolicy:**
- The AI Employee is the only pod allowed to initiate calls to other services
- No other pod can call the AI Employee directly from outside the namespace
- All outbound calls to external APIs go through dedicated integration pods

**Level 3 — Secret Rotation and Revocation:**
- If the AI Employee pod is compromised, its ServiceAccount can be
  immediately disabled by deleting the ServiceAccount token
- This instantly cuts off all access without affecting other components
- A new ServiceAccount and token can be issued after investigation
```yaml
# Emergency: revoke AI Employee access immediately
# Delete the ServiceAccount token to cut off all access
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
```
**Access reinstatement process:**
1. Complete security investigation
2. Rotate all Secrets the AI Employee had access to
3. Create a new ServiceAccount with a fresh token
4. Redeploy the AI Employee pod
5. Monitor logs for 24 hours after reinstatement

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
| RBAC resourceNames | All 4 components | Each pod can only read its own Secret |
| NetworkPolicy | All 4 components | Only AI Employee can initiate internal calls |
| Secret rotation | All Secrets | Every 60 days with expiry annotations |
| ServiceAccounts | All 4 components | No shared or default service accounts used |
