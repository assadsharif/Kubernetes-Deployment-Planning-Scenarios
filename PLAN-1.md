# Kubernetes Deployment Plan: AI Native Task Manager

**Date:** 2026-03-27
**Version:** 1.0
**Scenario:** AI Native Task Manager — a four-component AI-augmented task management platform.

---

## Architecture Overview

The system consists of four components: a **UI Interface** (frontend), **Backend APIs** (REST/gRPC), a **Todo Agent** (LLM-backed AI agent), and a **Notification API** (fan-out notification service).

Communication flows:
- UI ↔ Backend APIs via REST HTTPS (task CRUD operations)
- UI ↔ Todo Agent via REST HTTPS (AI-assisted task management)
- Todo Agent → Backend APIs via gRPC (programmatic task operations)
- Notification API ↔ UI via WebSocket push
- Notification API ↔ Backend APIs via REST

The mixed synchronous (REST) and agent-driven (LLM) communication patterns drive namespace separation and resource sizing decisions throughout this plan.

---

## 1. Namespace Design

### Namespaces

| Namespace | Purpose | Isolation Reason |
|---|---|---|
| `taskmanager-ui` | Frontend web application | Separate ingress-facing blast radius from backend |
| `taskmanager-api` | Backend REST/gRPC APIs | Data-layer isolation; independent RBAC and network policies |
| `taskmanager-agents` | Todo Agent (LLM) | AI workloads have unpredictable memory footprints; prevents resource starvation of API tier |
| `taskmanager-notifications` | Notification API | Independent scaling; isolated network egress for webhook delivery |

**Why not a single namespace?** The Todo Agent makes LLM inference calls that can spike memory consumption significantly. Co-locating it with the UI or API tier risks OOM evictions on shared nodes. Namespace-level `ResourceQuota` lets us give the agent tier a higher memory ceiling without affecting other tiers.

### Namespace YAMLs

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: taskmanager-ui
  labels:
    app.kubernetes.io/part-of: ai-task-manager
    tier: frontend
    environment: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: taskmanager-api
  labels:
    app.kubernetes.io/part-of: ai-task-manager
    tier: api
    environment: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: taskmanager-agents
  labels:
    app.kubernetes.io/part-of: ai-task-manager
    tier: ai-compute
    environment: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: taskmanager-notifications
  labels:
    app.kubernetes.io/part-of: ai-task-manager
    tier: notifications
    environment: production
```

### ResourceQuotas

**Notifications namespace (tight ceiling — lightweight service):**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: notifications-quota
  namespace: taskmanager-notifications
spec:
  hard:
    requests.cpu: "500m"
    requests.memory: "512Mi"
    limits.cpu: "1"
    limits.memory: "1Gi"
    pods: "10"
```

**AI Agents namespace (higher memory ceiling — LLM inference):**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: agents-quota
  namespace: taskmanager-agents
spec:
  hard:
    requests.cpu: "2"
    requests.memory: "4Gi"
    limits.cpu: "16"
    limits.memory: "16Gi"
    pods: "20"
```

---

## 2. Pods / Deployments / StatefulSets

**Design decision — all Deployments, no StatefulSets:** All four components are stateless per-request. The Backend APIs maintain application state in an external database (not local disk). The Todo Agent stores task context via Backend APIs (gRPC calls), not on local disk. Notification API is a pure fan-out relay. StatefulSets would add pod-identity complexity with no benefit here.

### 2.1 UI Interface

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui-interface
  namespace: taskmanager-ui
  labels:
    app: ui-interface
    app.kubernetes.io/part-of: ai-task-manager
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
        - name: ui
          image: registry/taskmanager-ui:latest
          ports:
            - containerPort: 3000
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 20
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

**2 replicas:** Provides HA without over-provisioning — UI is a thin static-serving layer.

### 2.2 Backend APIs

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  namespace: taskmanager-api
  labels:
    app: backend-api
    app.kubernetes.io/part-of: ai-task-manager
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend-api
  template:
    metadata:
      labels:
        app: backend-api
    spec:
      serviceAccountName: api-sa
      containers:
        - name: api
          image: registry/taskmanager-api:latest
          ports:
            - name: rest
              containerPort: 8080
            - name: grpc
              containerPort: 9090
          startupProbe:
            httpGet:
              path: /healthz
              port: 8080
            failureThreshold: 30
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 20
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"
          envFrom:
            - configMapRef:
                name: api-config
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: DB_PASSWORD
```

**3 replicas (odd number):** Odd replica counts avoid split-brain in gRPC client-side load balancing. `startupProbe` with 30 failure attempts (30×5s = 150s window) handles slow gRPC server initialization without false liveness failures.

### 2.3 Todo Agent

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-agent
  namespace: taskmanager-agents
  labels:
    app: todo-agent
    app.kubernetes.io/part-of: ai-task-manager
spec:
  replicas: 2
  selector:
    matchLabels:
      app: todo-agent
  template:
    metadata:
      labels:
        app: todo-agent
      annotations:
        reloader.stakater.com/auto: "true"
    spec:
      serviceAccountName: agent-sa
      terminationGracePeriodSeconds: 120
      containers:
        - name: agent
          image: registry/taskmanager-agent:latest
          ports:
            - containerPort: 8082
          readinessProbe:
            httpGet:
              path: /ready
              port: 8082
            initialDelaySeconds: 10
            periodSeconds: 15
          livenessProbe:
            httpGet:
              path: /live
              port: 8082
            initialDelaySeconds: 30
            periodSeconds: 30
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2000m"
              memory: "2Gi"
          envFrom:
            - configMapRef:
                name: agent-config
          env:
            - name: LLM_API_KEY
              valueFrom:
                secretKeyRef:
                  name: llm-api-secret
                  key: LLM_API_KEY
```

**`terminationGracePeriodSeconds: 120`:** LLM inference requests can run for 30–90 seconds. A 120-second grace period ensures in-flight requests complete before the pod is forcibly killed during rolling updates.

**HorizontalPodAutoscaler for Todo Agent:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: todo-agent-hpa
  namespace: taskmanager-agents
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

### 2.4 Notification API

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notification-api
  namespace: taskmanager-notifications
  labels:
    app: notification-api
    app.kubernetes.io/part-of: ai-task-manager
spec:
  replicas: 2
  selector:
    matchLabels:
      app: notification-api
  template:
    metadata:
      labels:
        app: notification-api
    spec:
      serviceAccountName: notification-sa
      containers:
        - name: notification
          image: registry/taskmanager-notifications:latest
          ports:
            - containerPort: 8081
          readinessProbe:
            httpGet:
              path: /health
              port: 8081
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8081
            initialDelaySeconds: 15
            periodSeconds: 20
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
          envFrom:
            - configMapRef:
                name: notification-config
          env:
            - name: NOTIFICATION_SIGNING_KEY
              valueFrom:
                secretKeyRef:
                  name: notification-secrets
                  key: SIGNING_KEY
```

---

## 3. Services and Service Types

| Service Name | Namespace | Type | Port(s) | Target | Rationale |
|---|---|---|---|---|---|
| `ui-service` | `taskmanager-ui` | LoadBalancer | 80, 443 | UI pods | Internet-facing; public entry point |
| `api-service` | `taskmanager-api` | ClusterIP | 8080 (REST), 9090 (gRPC) | API pods | Internal only; accessed by UI and Agent |
| `agent-service` | `taskmanager-agents` | ClusterIP | 8082 | Agent pods | Internal only; UI calls agent here |
| `notification-service` | `taskmanager-notifications` | ClusterIP | 8081 | Notification pods | Internal push/pull only |

**Why no NodePort?** In a multi-namespace production setup, NodePort port management (30000–32767 range) becomes brittle and error-prone. LoadBalancer handles public ingress for the UI; ClusterIP with DNS-based service discovery handles all internal communication.

### `api-service` YAML (dual-port ClusterIP):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: taskmanager-api
spec:
  type: ClusterIP
  selector:
    app: backend-api
  ports:
    - name: rest
      port: 8080
      targetPort: 8080
      protocol: TCP
    - name: grpc
      port: 9090
      targetPort: 9090
      protocol: TCP
```

### `ui-service` YAML (LoadBalancer):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ui-service
  namespace: taskmanager-ui
spec:
  type: LoadBalancer
  selector:
    app: ui-interface
  ports:
    - name: http
      port: 80
      targetPort: 3000
    - name: https
      port: 443
      targetPort: 3000
```

---

## 4. Resource Requests and Limits

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit | Sizing Rationale |
|---|---|---|---|---|---|
| UI Interface | 100m | 500m | 128Mi | 256Mi | Static file serving; very low compute overhead |
| Backend APIs | 250m | 1000m | 256Mi | 512Mi | DB queries + gRPC serialization; moderate burst |
| Todo Agent | 500m | 2000m | 512Mi | 2Gi | LLM context windows are memory-intensive; 4:1 limit-to-request ratio allows burst without hoarding |
| Notification API | 100m | 250m | 128Mi | 256Mi | Lightweight fan-out relay; bounded by message rate |

**4:1 limit-to-request ratio for Todo Agent explained:** The agent's baseline memory usage is ~512Mi (process + LLM client library). During active inference, large context windows can expand memory to ~1.5–2Gi. Setting a 4:1 ratio allows the Kubernetes scheduler to densely pack pods at request time while still allowing burst capacity on nodes that have headroom.

---

## 5. ConfigMaps for Configuration

**Principle:** ConfigMaps store non-sensitive configuration. Credentials, API keys, and signing secrets go in Secrets (see Section 6). One ConfigMap per namespace keeps coupling visible and avoids cross-namespace config dependencies.

### `api-config` (taskmanager-api namespace):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: taskmanager-api
data:
  DB_HOST: "postgres.taskmanager-data.svc.cluster.local"
  DB_PORT: "5432"
  DB_NAME: "taskmanager"
  GRPC_MAX_RECV_MSG_SIZE: "4194304"
  GRPC_KEEPALIVE_TIME_MS: "30000"
  FEATURE_AI_TASKS_ENABLED: "true"
  LOG_LEVEL: "info"
```

### `agent-config` (taskmanager-agents namespace):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: agent-config
  namespace: taskmanager-agents
data:
  LLM_PROVIDER_URL: "https://api.openai.com/v1"
  LLM_MODEL: "gpt-4o"
  LLM_MAX_TOKENS: "4096"
  LLM_TIMEOUT_SECONDS: "60"
  LLM_RETRY_ATTEMPTS: "3"
  BACKEND_API_GRPC_ENDPOINT: "api-service.taskmanager-api.svc.cluster.local:9090"
  LOG_LEVEL: "info"
```

### `notification-config` (taskmanager-notifications namespace):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: notification-config
  namespace: taskmanager-notifications
data:
  UI_WEBSOCKET_ENDPOINT: "ws://ui-service.taskmanager-ui.svc.cluster.local:3000/ws"
  API_ENDPOINT: "http://api-service.taskmanager-api.svc.cluster.local:8080"
  WEBHOOK_RETRY_INTERVAL_SECONDS: "30"
  WEBHOOK_MAX_RETRIES: "5"
  LOG_LEVEL: "info"
```

---

## 6. Secrets Management

### 6.1 ConfigMap vs Secret Decision Matrix

| Configuration Item | ConfigMap | Secret | Reason |
|---|---|---|---|
| LLM provider URL | X | | Non-sensitive endpoint URL |
| LLM API key | | X | Credential; encrypted at rest in etcd |
| Database host/port | X | | Non-sensitive connection info |
| Database password | | X | Credential |
| Notification webhook template URL | X | | URL itself is not a secret |
| Notification payload signing key | | X | HMAC secret used to authenticate payloads |

### 6.2 Secret Definitions

**LLM API key (taskmanager-agents namespace):**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: llm-api-secret
  namespace: taskmanager-agents
  annotations:
    secret-expiry-date: "2026-09-27"
    secret-rotation-policy: "90-days"
    secret-owner: "platform-team"
type: Opaque
data:
  LLM_API_KEY: <base64-encoded-api-key>
```

**Database credentials (taskmanager-api namespace):**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: taskmanager-api
  annotations:
    secret-expiry-date: "2026-09-27"
    secret-rotation-policy: "90-days"
type: Opaque
data:
  DB_PASSWORD: <base64-encoded-password>
```

### 6.3 ExternalSecrets Operator (Recommended Production Pattern)

Rather than managing K8s Secrets directly, use the **ExternalSecrets Operator (ESO)** with an external secret store (Vault, AWS Secrets Manager, or GCP Secret Manager). The K8s Secret becomes a synced copy — the external store is the authoritative source.

**SecretStore definition (Vault backend):**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-secret-store
  namespace: taskmanager-agents
spec:
  provider:
    vault:
      server: "https://vault.internal:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "taskmanager-agent-role"
```

**ExternalSecret for LLM API key:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: llm-api-external-secret
  namespace: taskmanager-agents
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-secret-store
    kind: SecretStore
  target:
    name: llm-api-secret
    creationPolicy: Owner
  data:
    - secretKey: LLM_API_KEY
      remoteRef:
        key: taskmanager/llm
        property: api_key
```

### 6.4 Secret Rotation Flow

```
1. Platform team updates credential in Vault
2. ESO detects change on next refreshInterval (≤ 1h)
3. K8s Secret is updated by ESO controller
4. Reloader annotation on Deployment detects Secret change
5. Rolling update begins automatically — new pods get updated credential
6. Old pods drain gracefully within terminationGracePeriodSeconds
```

### 6.5 Compromised Secret Response Procedure

1. **Revoke** the credential immediately at the provider (OpenAI dashboard, database IAM)
2. **Force ESO sync** to avoid waiting for the next refresh interval:
   ```bash
   kubectl annotate externalsecret llm-api-external-secret \
     force-sync=$(date +%s) -n taskmanager-agents
   ```
3. **Rolling restart** once the new credential is provisioned in Vault:
   ```bash
   kubectl rollout restart deployment/todo-agent -n taskmanager-agents
   ```
4. **Audit**: Record revocation timestamp, affected pod names (`kubectl get pods -n taskmanager-agents`), and the new Vault secret version
5. **Verify**: Test that the new credential works with a smoke-test request before closing the incident
6. **Post-mortem**: Update the `secret-expiry-date` annotation on the ExternalSecret to reflect the new rotation schedule

---

## 7. RBAC Roles and RoleBindings

### 7.1 ServiceAccounts (Principle of Least Privilege)

One ServiceAccount per Deployment. No Deployment shares a ServiceAccount. None are granted `cluster-admin`.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ui-sa
  namespace: taskmanager-ui
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-sa
  namespace: taskmanager-api
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: agent-sa
  namespace: taskmanager-agents
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: notification-sa
  namespace: taskmanager-notifications
```

### 7.2 Todo Agent Role (taskmanager-agents namespace)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: agent-role
  namespace: taskmanager-agents
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["llm-api-secret"]
    verbs: ["get"]
```

**`resourceNames` restriction:** This limits the `agent-sa` ServiceAccount to reading only the `llm-api-secret` — not all Secrets in the namespace. Without this restriction, a compromised agent pod could enumerate and exfiltrate all namespace secrets.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: agent-role-binding
  namespace: taskmanager-agents
subjects:
  - kind: ServiceAccount
    name: agent-sa
    namespace: taskmanager-agents
roleRef:
  kind: Role
  name: agent-role
  apiGroup: rbac.authorization.k8s.io
```

### 7.3 Backend API Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: api-role
  namespace: taskmanager-api
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["db-credentials"]
    verbs: ["get"]
```

### 7.4 Cross-Namespace Access Note

> **RBAC does not restrict pod-to-pod network calls across namespaces.** The Todo Agent calling `api-service.taskmanager-api.svc.cluster.local:9090` is governed by NetworkPolicy (Section 8), not RBAC. RBAC only controls access to the Kubernetes API server resources (Secrets, ConfigMaps, Pods, etc.).

---

## 8. Inter-Service Communication

### 8.1 Communication Map

| Source | Destination | Protocol | Service DNS | Auth Method |
|---|---|---|---|---|
| UI Interface | Backend APIs | REST HTTPS | `api-service.taskmanager-api.svc.cluster.local:8080` | JWT Bearer token |
| UI Interface | Todo Agent | REST HTTPS | `agent-service.taskmanager-agents.svc.cluster.local:8082` | JWT Bearer token |
| Todo Agent | Backend APIs | gRPC | `api-service.taskmanager-api.svc.cluster.local:9090` | Service account token + mTLS |
| Notification API | UI Interface | WebSocket | `ui-service.taskmanager-ui.svc.cluster.local:3000` | HMAC signing key (from Secret) |
| Notification API | Backend APIs | REST | `api-service.taskmanager-api.svc.cluster.local:8080` | JWT Bearer token |

### 8.2 NetworkPolicy for Todo Agent

Allows only the egress the agent legitimately needs: gRPC to the API tier, HTTPS to the external LLM provider. Denies all other egress (agent cannot initiate calls to the notification or UI tiers).

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: agent-egress-policy
  namespace: taskmanager-agents
spec:
  podSelector:
    matchLabels:
      app: todo-agent
  policyTypes:
    - Egress
  egress:
    # Allow gRPC to Backend APIs namespace
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: taskmanager-api
      ports:
        - port: 9090
          protocol: TCP
    # Allow HTTPS to external LLM API (e.g., OpenAI)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - port: 443
          protocol: TCP
    # Allow DNS resolution
    - ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
```

### 8.3 NetworkPolicy for Notification API

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: notification-egress-policy
  namespace: taskmanager-notifications
spec:
  podSelector:
    matchLabels:
      app: notification-api
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: taskmanager-ui
      ports:
        - port: 3000
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: taskmanager-api
      ports:
        - port: 8080
    - ports:
        - port: 53
          protocol: UDP
```

---

## 9. Open Questions / Next Steps

The following assumptions were made in this plan. Resolve these before committing to production:

1. **Database location**: This plan assumes an external database (referenced via `DB_HOST` in ConfigMap). If running Postgres in-cluster, add a StatefulSet + PVC for the database tier and a dedicated `taskmanager-data` namespace.

2. **LLM provider**: Assumes external OpenAI-compatible API (`LLM_PROVIDER_URL`). If self-hosting a model, add GPU node sizing from `K8-PLANNING-SKILL/references/resource-sizing.md` and a dedicated `taskmanager-llm` namespace.

3. **TLS termination**: The `ui-service` LoadBalancer exposes port 443 but TLS termination configuration depends on your cloud provider (GKE, EKS, AKS) or Ingress controller. Add an Ingress resource with cert-manager if using nginx-ingress.

4. **Image registry**: Replace all `registry/` image prefixes with your actual container registry path.

5. **Reloader**: The `reloader.stakater.com/auto: "true"` annotation on the Todo Agent Deployment requires [Stakater Reloader](https://github.com/stakater/Reloader) to be installed. Without it, secret changes won't trigger automatic pod restarts.
