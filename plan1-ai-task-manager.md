# Plan 1: Kubernetes Deployment Plan — AI Native Task Manager

## Application Overview

| Component | Role | External Access |
|---|---|---|
| UI Interface | Frontend dashboard for users | Yes — via Ingress |
| Backend APIs | Core business logic and data layer | No — internal only |
| Todo Agent | AI-powered task processing worker | No — internal only |
| Notification Service | Sends alerts via UI and Backend | No — internal only |

---

## 1. Namespace

All components run inside a dedicated namespace for isolation.
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: task-manager
```

> **Why:** Namespaces isolate resources, prevent naming conflicts, and allow
> separate RBAC policies per environment.

---

## 2. Deployments

All four components are stateless, so `Deployment` is used (not `StatefulSet`).

### UI Interface
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui-interface
  namespace: task-manager
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ui-interface
  template:
    metadata:
      labels:
        app: ui-interface
    spec:
      serviceAccountName: ui-sa
      containers:
      - name: ui-interface
        image: ui-interface:latest
        ports:
        - containerPort: 3000
        envFrom:
        - configMapRef:
            name: app-config
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "300m"
            memory: "256Mi"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Backend APIs
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-apis
  namespace: task-manager
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend-apis
  template:
    metadata:
      labels:
        app: backend-apis
    spec:
      serviceAccountName: backend-sa
      containers:
      - name: backend-apis
        image: backend-apis:latest
        ports:
        - containerPort: 8080
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets
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
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Todo Agent
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-agent
  namespace: task-manager
spec:
  replicas: 2
  selector:
    matchLabels:
      app: todo-agent
  template:
    metadata:
      labels:
        app: todo-agent
    spec:
      serviceAccountName: todo-agent-sa
      containers:
      - name: todo-agent
        image: todo-agent:latest
        ports:
        - containerPort: 5000
        envFrom:
        - configMapRef:
            name: app-config
        resources:
          requests:
            cpu: "300m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 15
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 5
```

### Notification Service
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notification-service
  namespace: task-manager
spec:
  replicas: 2
  selector:
    matchLabels:
      app: notification-service
  template:
    metadata:
      labels:
        app: notification-service
    spec:
      serviceAccountName: notification-sa
      containers:
      - name: notification-service
        image: notification-service:latest
        ports:
        - containerPort: 4000
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: notification-secret
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "300m"
            memory: "256Mi"
        livenessProbe:
          httpGet:
            path: /health
            port: 4000
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 4000
          initialDelaySeconds: 5
          periodSeconds: 5
```

---

## 3. Services

All services use `ClusterIP` for internal communication. External access is
handled centrally by the Ingress Controller.

| Component | Service Type | Port | Reason |
|---|---|---|---|
| UI Interface | ClusterIP | 80 | External access via Ingress |
| Backend APIs | ClusterIP | 8080 | Internal only |
| Todo Agent | ClusterIP | 5000 | Internal only |
| Notification Service | ClusterIP | 4000 | Internal only |
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ui-interface-svc
  namespace: task-manager
spec:
  type: ClusterIP
  selector:
    app: ui-interface
  ports:
  - port: 80
    targetPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: backend-apis-svc
  namespace: task-manager
spec:
  type: ClusterIP
  selector:
    app: backend-apis
  ports:
  - port: 8080
    targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: todo-agent-svc
  namespace: task-manager
spec:
  type: ClusterIP
  selector:
    app: todo-agent
  ports:
  - port: 5000
    targetPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: notification-svc
  namespace: task-manager
spec:
  type: ClusterIP
  selector:
    app: notification-service
  ports:
  - port: 4000
    targetPort: 4000
```

---

## 4. Ingress — Production Access Layer

Instead of exposing the UI directly via LoadBalancer, an Ingress Controller
is used for centralized routing, SSL termination, and better production control.
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ui-ingress
  namespace: task-manager
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  rules:
  - host: taskmanager.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ui-interface-svc
            port:
              number: 80
  tls:
  - hosts:
    - taskmanager.example.com
    secretName: ui-tls-secret
```

**Why Ingress instead of LoadBalancer:**

| Reason | Detail |
|---|---|
| Single entry point | All external traffic goes through one controller |
| SSL/TLS termination | HTTPS handled centrally |
| Host based routing | Easy to add new routes without new Services |
| Cost efficient | Only one cloud load balancer needed |
| Production standard | Industry standard for production deployments |

---

## 5. ConfigMaps

Non-sensitive configuration stored as ConfigMaps.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: task-manager
data:
  BACKEND_API_URL: "http://backend-apis-svc:8080"
  TODO_AGENT_URL: "http://todo-agent-svc:5000"
  NOTIFICATION_URL: "http://notification-svc:4000"
  LOG_LEVEL: "info"
  ENVIRONMENT: "production"
  MAX_TASKS_PER_USER: "100"
```

---

## 6. Secrets

Sensitive data stored as Kubernetes Secrets. Never stored in ConfigMaps or
source code.
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: task-manager
  annotations:
    secret-expiry-date: "2025-12-31"
    rotation-schedule: "every-90-days"
type: Opaque
data:
  DB_PASSWORD: <base64-encoded-value>
  API_KEY: <base64-encoded-value>
  JWT_SECRET: <base64-encoded-value>
---
apiVersion: v1
kind: Secret
metadata:
  name: notification-secret
  namespace: task-manager
  annotations:
    secret-expiry-date: "2025-12-31"
    rotation-schedule: "every-90-days"
type: Opaque
data:
  NOTIFICATION_API_KEY: <base64-encoded-value>
---
apiVersion: v1
kind: Secret
metadata:
  name: ui-tls-secret
  namespace: task-manager
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
```

**Secret Management Rules:**
- Secrets are annotated with `secret-expiry-date` for rotation tracking
- Each component accesses only its own Secret — no shared master secret
- Secrets rotated every 90 days
- TLS Secret managed separately for Ingress SSL termination
- In production: integrate with HashiCorp Vault or AWS Secrets Manager

**Additional Security Practices:**
- Secrets are injected as environment variables or mounted as volumes
- Secrets are never logged or exposed in application responses
- Access is strictly controlled via RBAC policies

---

## 7. RBAC — Roles and RoleBindings

Each component runs under its own ServiceAccount with minimum required
permissions (least privilege principle).
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ui-sa
  namespace: task-manager
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend-sa
  namespace: task-manager
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: todo-agent-sa
  namespace: task-manager
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: notification-sa
  namespace: task-manager
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: backend-role
  namespace: task-manager
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-rolebinding
  namespace: task-manager
subjects:
- kind: ServiceAccount
  name: backend-sa
  namespace: task-manager
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: backend-role
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: todo-agent-role
  namespace: task-manager
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: todo-agent-rolebinding
  namespace: task-manager
subjects:
- kind: ServiceAccount
  name: todo-agent-sa
  namespace: task-manager
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: todo-agent-role
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: notification-role
  namespace: task-manager
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: notification-rolebinding
  namespace: task-manager
subjects:
- kind: ServiceAccount
  name: notification-sa
  namespace: task-manager
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: notification-role
```

---

## 8. Inter-Service Communication

The UI does not directly communicate with the Todo Agent to maintain proper
abstraction, security, and centralized control through the Backend APIs.
```
Internet
    │
    ▼
Ingress Controller (taskmanager.example.com HTTPS:443)
    │
    ▼
UI Interface (ClusterIP :80)
    │
    └──► Backend APIs (ClusterIP :8080)
               │
               ├──► Todo Agent (ClusterIP :5000)
               │
               └──► Notification Service (ClusterIP :4000)
                          │
                          └──► UI Interface (ClusterIP :3000)
```

| From | To | Protocol | Port | Reason |
|---|---|---|---|---|
| Internet | Ingress Controller | HTTPS | 443 | External entry point |
| Ingress Controller | UI Interface | HTTP | 80 | Internal routing |
| UI Interface | Backend APIs | HTTP | 8080 | All client requests go through backend |
| Backend APIs | Todo Agent | HTTP | 5000 | Backend delegates AI processing |
| Backend APIs | Notification Service | HTTP | 4000 | Backend triggers alerts |
| Notification Service | UI Interface | HTTP | 3000 | Push updates to UI |

> **Note:** UI does not directly communicate with Todo Agent. All requests
> go through Backend APIs to maintain proper layered architecture and
> centralized security control.

---

## 9. Network Policies — Zero-Trust Security

Network policies restrict communication between services to only what is
necessary — no component can talk to another unless explicitly allowed.
```yaml
# Backend APIs — only accepts from UI and Notification Service
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: task-manager
spec:
  podSelector:
    matchLabels:
      app: backend-apis
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: ui-interface
    - podSelector:
        matchLabels:
          app: notification-service
  policyTypes:
  - Ingress
---
# Todo Agent — only accepts from Backend APIs
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: todo-agent-policy
  namespace: task-manager
spec:
  podSelector:
    matchLabels:
      app: todo-agent
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend-apis
  policyTypes:
  - Ingress
---
# Notification Service — only accepts from Backend APIs
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: notification-policy
  namespace: task-manager
spec:
  podSelector:
    matchLabels:
      app: notification-service
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend-apis
  policyTypes:
  - Ingress
```

**Policy Rules:**
- UI → Backend APIs ✅ allowed
- Notification → Backend APIs ✅ allowed
- Backend APIs → Todo Agent ✅ allowed
- Backend APIs → Notification Service ✅ allowed
- Notification Service → UI Interface ✅ allowed
- UI → Todo Agent ❌ blocked
- Any other direct pod-to-pod communication ❌ blocked

---

## 10. HorizontalPodAutoscaler — Todo Agent

The Todo Agent handles AI workloads that spike unpredictably. HPA scales
it automatically based on CPU usage.
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: todo-agent-hpa
  namespace: task-manager
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: todo-agent
  minReplicas: 2
  maxReplicas: 8
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

## 11. Failure Handling and Reliability

The system is designed for resilience and high availability across all
components.

| Feature | How It Works | Benefit |
|---|---|---|
| Liveness Probes | Kubernetes restarts crashed containers automatically | Self-healing |
| Readiness Probes | Traffic only sent to healthy and ready pods | No failed requests |
| Multiple Replicas | All components run minimum 2 replicas | No single point of failure |
| HPA on Todo Agent | Auto-scales from 2 to 8 replicas based on CPU | Handles load spikes |
| Ingress TLS | SSL termination at entry point | Secure external access |
| Namespace Isolation | All components isolated from other apps | Blast radius reduction |
| NetworkPolicies | Zero-trust pod-to-pod communication | Security in depth |

**Result:**
- High availability — no single point of failure
- Fault tolerance — crashed containers auto-recover
- Zero-downtime deployments — readiness probes prevent bad rollouts
- Auto-scaling — system handles traffic spikes without manual intervention

---

## Summary Table

| Component | Type | Replicas | Service Type | CPU Limit | Memory Limit |
|---|---|---|---|---|---|
| UI Interface | Deployment | 2 | ClusterIP + Ingress | 300m | 256Mi |
| Backend APIs | Deployment | 3 | ClusterIP | 500m | 512Mi |
| Todo Agent | Deployment | 2–8 (HPA) | ClusterIP | 1000m | 1Gi |
| Notification Service | Deployment | 2 | ClusterIP | 300m | 256Mi |
| Ingress Controller | nginx | — | External Entry Point | — | — |
