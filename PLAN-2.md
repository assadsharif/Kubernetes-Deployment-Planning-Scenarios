# Kubernetes Deployment Plan: AI Employee using OpenClaw

**Date:** 2026-03-27
**Version:** 1.0
**Scenario:** Personal AI Employee — OpenClaw agent integrated with Google Workspace, WhatsApp, Discord, and an LLM Service.

---

## Architecture Overview

OpenClaw is a persistent AI agent that acts as a **personal employee**: it reads and acts on emails and calendar events (Google Workspace), communicates via messaging platforms (WhatsApp, Discord), and uses an LLM service for reasoning and response generation.

Unlike the AI Task Manager (Plan-1), this system processes **sensitive personal and organizational data** — email content, calendar entries, private messages. The security posture is therefore the dominant design driver:

- Secrets management is more granular (6 distinct credential types)
- RBAC is explicitly designed to prevent the AI agent from accessing integration credentials (limiting exfiltration scope)
- Cross-namespace communication uses mTLS recommendation
- An operational runbook covers compromised-secret response

**External Integration Sources:**
- Google Workspace (push notifications for Gmail, Calendar)
- WhatsApp Business API (webhook push)
- Discord (gateway WebSocket connection)

---

## 1. Namespace Design

### Namespaces

| Namespace | Purpose | Isolation Reason |
|---|---|---|
| `openclaw-core` | OpenClaw AI Employee agent | Central orchestrator; isolated for independent resource control |
| `openclaw-integrations` | Google Workspace, WhatsApp, Discord connectors | Outbound-only integrations; separate network egress policies |
| `openclaw-llm` | LLM Service | Isolated for GPU scheduling; independent scaling from agent tier |
| `openclaw-secrets-mgmt` | ExternalSecrets Operator | ESO controller has elevated permissions; blast-radius isolation is critical |

**Why a dedicated `openclaw-secrets-mgmt` namespace?** The ExternalSecrets Operator controller must be able to read from Vault/cloud secret stores and write synced K8s Secrets across namespaces. This elevated permission set must be tightly scoped. Isolating ESO in its own namespace means you can apply strict RBAC to the `eso-controller-sa` without that permission bleeding into application namespaces.

### Namespace YAMLs

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openclaw-core
  labels:
    app.kubernetes.io/part-of: openclaw-ai-employee
    tier: agent
    environment: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: openclaw-integrations
  labels:
    app.kubernetes.io/part-of: openclaw-ai-employee
    tier: integrations
    environment: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: openclaw-llm
  labels:
    app.kubernetes.io/part-of: openclaw-ai-employee
    tier: ai-compute
    environment: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: openclaw-secrets-mgmt
  labels:
    app.kubernetes.io/part-of: openclaw-ai-employee
    tier: infrastructure
    environment: production
```

### ResourceQuotas

**Core agent namespace:**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: core-quota
  namespace: openclaw-core
spec:
  hard:
    requests.cpu: "2"
    requests.memory: "4Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "10"
```

**Integrations namespace:**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: integrations-quota
  namespace: openclaw-integrations
spec:
  hard:
    requests.cpu: "1"
    requests.memory: "1Gi"
    limits.cpu: "4"
    limits.memory: "4Gi"
    pods: "20"
```

---

## 2. Pods / Deployments / StatefulSets

### 2.1 OpenClaw AI Employee Agent

**State management trade-off:**

| Option | When to Use | Trade-off |
|---|---|---|
| Deployment + Redis/external store | Multiple agent replicas, shared session state | Requires Redis; agent context serialized to Redis on each turn |
| StatefulSet + PVC | Single agent with long-running local context | Simpler wiring; but pod-identity coupling and slower rollouts |

**Recommended for initial deployment:** Deployment (1 replica) with session state externalized. Easier to scale and roll out updates.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openclaw-agent
  namespace: openclaw-core
  labels:
    app: openclaw-agent
    app.kubernetes.io/part-of: openclaw-ai-employee
spec:
  replicas: 1
  selector:
    matchLabels:
      app: openclaw-agent
  template:
    metadata:
      labels:
        app: openclaw-agent
      annotations:
        reloader.stakater.com/auto: "true"
    spec:
      serviceAccountName: openclaw-agent-sa
      terminationGracePeriodSeconds: 180
      containers:
        - name: agent
          image: registry/openclaw-agent:latest
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /live
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 30
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2000m"
              memory: "4Gi"
          envFrom:
            - configMapRef:
                name: openclaw-agent-config
          env:
            - name: LLM_API_KEY
              valueFrom:
                secretKeyRef:
                  name: llm-api-key
                  key: LLM_API_KEY
          volumeMounts:
            - name: system-prompt
              mountPath: /etc/openclaw/prompts
              readOnly: true
      volumes:
        - name: system-prompt
          configMap:
            name: openclaw-agent-config
            items:
              - key: SYSTEM_PROMPT_FILE
                path: system-prompt.txt
```

**PodDisruptionBudget** (prevents eviction during node drain while agent is active):
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: openclaw-agent-pdb
  namespace: openclaw-core
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: openclaw-agent
```

### 2.2 Google Workspace Connector

OAuth tokens expire every hour. A **token-refresh sidecar** handles continuous token renewal without restarting the main container.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: google-workspace-connector
  namespace: openclaw-integrations
  labels:
    app: google-workspace-connector
spec:
  replicas: 2
  selector:
    matchLabels:
      app: google-workspace-connector
  template:
    metadata:
      labels:
        app: google-workspace-connector
    spec:
      serviceAccountName: google-workspace-sa
      containers:
        # Main connector: reads email/calendar, routes to OpenClaw agent
        - name: connector
          image: registry/google-workspace-connector:latest
          ports:
            - containerPort: 9090
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          env:
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: /var/secrets/google/service-account.json
          volumeMounts:
            - name: google-service-account
              mountPath: /var/secrets/google
              readOnly: true
            - name: token-cache
              mountPath: /var/tokens

        # Sidecar: exchanges refresh token for access token every 45 minutes
        - name: token-refresher
          image: registry/token-refresher:latest
          resources:
            requests:
              cpu: "10m"
              memory: "32Mi"
            limits:
              cpu: "50m"
              memory: "64Mi"
          env:
            - name: GOOGLE_REFRESH_TOKEN
              valueFrom:
                secretKeyRef:
                  name: google-oauth-tokens
                  key: REFRESH_TOKEN
            - name: TOKEN_REFRESH_INTERVAL: "2700"  # 45 minutes
            - name: TOKEN_OUTPUT_PATH: "/var/tokens/access_token"
          volumeMounts:
            - name: token-cache
              mountPath: /var/tokens

      volumes:
        # Service account JSON from Secret (mounted as file, not env var — prevents accidental logging)
        - name: google-service-account
          secret:
            secretName: google-service-account
        # Shared emptyDir: sidecar writes token, main container reads it
        - name: token-cache
          emptyDir: {}
```

**Why mount service account JSON as a file, not an environment variable?** Environment variables appear in process listings, crash dumps, and container logs. Mounting as a volume file limits exposure to the filesystem, and access can be audited at the file level.

### 2.3 WhatsApp Connector

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whatsapp-connector
  namespace: openclaw-integrations
  labels:
    app: whatsapp-connector
spec:
  replicas: 2
  selector:
    matchLabels:
      app: whatsapp-connector
  template:
    metadata:
      labels:
        app: whatsapp-connector
    spec:
      serviceAccountName: whatsapp-connector-sa
      containers:
        - name: whatsapp
          image: registry/whatsapp-connector:latest
          ports:
            - containerPort: 8090
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "300m"
              memory: "256Mi"
          envFrom:
            - configMapRef:
                name: integration-config
          env:
            - name: WHATSAPP_API_TOKEN
              valueFrom:
                secretKeyRef:
                  name: whatsapp-api-token
                  key: API_TOKEN
            - name: WEBHOOK_SIGNING_SECRET
              valueFrom:
                secretKeyRef:
                  name: webhook-signing-secret
                  key: SIGNING_SECRET
```

### 2.4 Discord Connector

Discord uses a persistent WebSocket gateway connection — a single long-lived connection to Discord's gateway. **1 replica** is appropriate; horizontal scaling of gateway connections requires Discord bot sharding logic (out of scope here).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: discord-connector
  namespace: openclaw-integrations
  labels:
    app: discord-connector
spec:
  replicas: 1
  selector:
    matchLabels:
      app: discord-connector
  template:
    metadata:
      labels:
        app: discord-connector
    spec:
      serviceAccountName: discord-connector-sa
      terminationGracePeriodSeconds: 30
      containers:
        - name: discord
          image: registry/discord-connector:latest
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "300m"
              memory: "512Mi"
          envFrom:
            - configMapRef:
                name: integration-config
          env:
            - name: DISCORD_BOT_TOKEN
              valueFrom:
                secretKeyRef:
                  name: discord-bot-token
                  key: BOT_TOKEN
```

### 2.5 LLM Service

**Option A: External LLM API (recommended for initial deployment)**

No Deployment needed. Use an `ExternalName` Service to route internal calls to the external API:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: llm-service
  namespace: openclaw-llm
spec:
  type: ExternalName
  externalName: api.openai.com
```

**Option B: Self-hosted LLM (GPU-accelerated)**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llm-service
  namespace: openclaw-llm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: llm-service
  template:
    metadata:
      labels:
        app: llm-service
    spec:
      nodeSelector:
        cloud.google.com/gke-accelerator: nvidia-tesla-t4
      containers:
        - name: llm
          image: registry/llm-server:latest
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "4000m"
              memory: "8Gi"
            limits:
              cpu: "8000m"
              memory: "16Gi"
              nvidia.com/gpu: "1"
          volumeMounts:
            - name: model-weights
              mountPath: /models
      volumes:
        - name: model-weights
          persistentVolumeClaim:
            claimName: model-weights-pvc
```

---

## 3. Services and Service Types

| Service Name | Namespace | Type | Port | Rationale |
|---|---|---|---|---|
| `openclaw-agent-service` | `openclaw-core` | ClusterIP | 8080 | Internal orchestration only; no external access needed |
| `google-workspace-service` | `openclaw-integrations` | ClusterIP | 9090 | Internal; Google push notifications route through cluster ingress, not direct pod access |
| `whatsapp-webhook-service` | `openclaw-integrations` | LoadBalancer | 8090 | WhatsApp requires a publicly reachable HTTPS webhook endpoint |
| `discord-connector-service` | `openclaw-integrations` | ClusterIP | (none needed) | Discord connector makes outbound WebSocket only; no inbound service required |
| `llm-service` | `openclaw-llm` | ClusterIP (or ExternalName) | 8080 / 443 | Internal if self-hosted; ExternalName if external API |

### Ingress for WhatsApp Webhook

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whatsapp-webhook-ingress
  namespace: openclaw-integrations
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
    - hosts:
        - webhook.yourdomain.com
      secretName: whatsapp-webhook-tls
  rules:
    - host: webhook.yourdomain.com
      http:
        paths:
          - path: /whatsapp
            pathType: Prefix
            backend:
              service:
                name: whatsapp-webhook-service
                port:
                  number: 8090
          - path: /google
            pathType: Prefix
            backend:
              service:
                name: google-workspace-service
                port:
                  number: 9090
```

---

## 4. Resource Requests and Limits

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit | Notes |
|---|---|---|---|---|---|
| OpenClaw Agent | 500m | 2000m | 1Gi | 4Gi | Long-context reasoning; large LLM response processing |
| Google Workspace Connector | 100m | 500m | 128Mi | 256Mi | API polling + push notification relay |
| Google Token-Refresh Sidecar | 10m | 50m | 32Mi | 64Mi | Lightweight background process |
| WhatsApp Connector | 100m | 300m | 128Mi | 256Mi | Webhook relay |
| Discord Connector | 100m | 300m | 256Mi | 512Mi | WebSocket connection overhead |
| LLM Service (self-hosted) | 4000m | 8000m | 8Gi | 16Gi | GPU workload; must run on GPU node pool |

---

## 5. ConfigMaps

### `openclaw-agent-config` (openclaw-core namespace):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: openclaw-agent-config
  namespace: openclaw-core
data:
  LLM_SERVICE_ENDPOINT: "http://llm-service.openclaw-llm.svc.cluster.local:8080"
  LLM_MODEL: "gpt-4o"
  LLM_MAX_TOKENS: "8192"
  # Path to system prompt file (mounted from volume)
  # Storing large prompts as a file prevents them from being logged
  SYSTEM_PROMPT_FILE: "/etc/openclaw/prompts/system-prompt.txt"
  TASK_QUEUE_POLL_INTERVAL_SECONDS: "10"
  GOOGLE_CONNECTOR_ENDPOINT: "http://google-workspace-service.openclaw-integrations.svc.cluster.local:9090"
  WHATSAPP_CONNECTOR_ENDPOINT: "http://whatsapp-webhook-service.openclaw-integrations.svc.cluster.local:8090"
  DISCORD_CONNECTOR_ENDPOINT: "http://discord-connector-service.openclaw-integrations.svc.cluster.local:8091"
  LOG_LEVEL: "info"
```

**Why store the system prompt path (not content) in ConfigMap?** System prompts can be multi-kilobyte strings. Storing them inline in environment variables risks accidental logging of the full prompt text. Mounting as a volume file allows the agent to read it at startup without it appearing in `kubectl describe pod` or container logs.

### `integration-config` (openclaw-integrations namespace):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: integration-config
  namespace: openclaw-integrations
data:
  # Google Workspace
  GOOGLE_WORKSPACE_SCOPES: "https://www.googleapis.com/auth/gmail.readonly,https://www.googleapis.com/auth/calendar"
  GOOGLE_API_BASE_URL: "https://www.googleapis.com"
  # WhatsApp
  WHATSAPP_API_VERSION: "v18.0"
  WHATSAPP_API_BASE_URL: "https://graph.facebook.com"
  WHATSAPP_MESSAGE_RATE_LIMIT: "80"
  # Discord
  DISCORD_GATEWAY_INTENTS: "33280"
  DISCORD_API_BASE_URL: "https://discord.com/api/v10"
  DISCORD_MESSAGE_RATE_LIMIT: "50"
  # OpenClaw agent endpoint (for routing replies back)
  AGENT_ENDPOINT: "http://openclaw-agent-service.openclaw-core.svc.cluster.local:8080"
```

**Rationale for splitting configs:** If you swap the LLM provider (e.g., OpenAI → Anthropic), you only update `openclaw-agent-config` and redeploy the agent. Integration connector configuration (API versions, rate limits, scopes) is stable and does not change with LLM provider changes.

---

## 6. Secrets Management and Expiry Handling

This is the most critical section for an AI Personal Employee because a credential compromise can expose the user's emails, calendar, and private messages.

### 6.1 Secret Inventory

| Secret Name | Namespace | Credential Type | Expiry Window | Source |
|---|---|---|---|---|
| `llm-api-key` | `openclaw-core` | API key | 90 days | Vault / AWS Secrets Manager |
| `google-service-account` | `openclaw-integrations` | GCP service account JSON | Managed by GCP | GCP Secret Manager |
| `google-oauth-tokens` | `openclaw-integrations` | OAuth refresh token | ~6 months (refresh); access token: 1hr (managed by sidecar) | GCP Secret Manager |
| `whatsapp-api-token` | `openclaw-integrations` | Bearer token | 60 days | Meta Developer Portal |
| `discord-bot-token` | `openclaw-integrations` | Bot token | Never (until manually revoked) | Discord Developer Portal |
| `webhook-signing-secret` | `openclaw-integrations` | HMAC signing secret | 180 days | Vault |

### 6.2 ExternalSecrets Operator Configuration

**SecretStore (Vault with Kubernetes auth):**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-secret-store
  namespace: openclaw-core
spec:
  provider:
    vault:
      server: "https://vault.internal:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "openclaw-agent-vault-role"
```

**ExternalSecret for LLM API key:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: llm-api-key-external
  namespace: openclaw-core
  annotations:
    secret-rotation-policy: "90-days"
    secret-owner: "platform-team"
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-secret-store
    kind: SecretStore
  target:
    name: llm-api-key
    creationPolicy: Owner
  data:
    - secretKey: LLM_API_KEY
      remoteRef:
        key: openclaw/llm
        property: api_key
```

**ExternalSecret for WhatsApp API token (60-day expiry, more aggressive refresh):**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: whatsapp-api-token-external
  namespace: openclaw-integrations
  annotations:
    secret-expiry-date: "2026-05-27"
    secret-rotation-policy: "60-days"
spec:
  refreshInterval: 30m
  secretStoreRef:
    name: vault-secret-store
    kind: SecretStore
  target:
    name: whatsapp-api-token
    creationPolicy: Owner
  data:
    - secretKey: API_TOKEN
      remoteRef:
        key: openclaw/whatsapp
        property: api_token
```

### 6.3 Compromised or Expired Secret — Operational Runbook

**Use this procedure when:** a secret is compromised (leaked, suspicious access), a rotation is overdue, or a provider forces revocation.

1. **Detection**
   - Monitoring alert fires: secret age exceeds `secret-expiry-date` annotation, or the provider (Meta, Google, OpenAI, Discord) sends a compromise notification
   - Confirm which secret is affected by checking the annotation: `kubectl get externalsecret -A -o jsonpath='{range .items[*]}{.metadata.name}: {.metadata.annotations.secret-expiry-date}{"\n"}{end}'`

2. **Immediate Revocation**
   - Revoke the credential at the issuing provider:
     - OpenAI/LLM: OpenAI dashboard → API Keys → Revoke
     - WhatsApp: Meta Developer Portal → Access Tokens → Invalidate
     - Google service account: GCP IAM → Service Accounts → Disable key
     - Discord bot: Discord Developer Portal → Bot → Reset Token
   - Document the revocation time in your incident log

3. **Force ESO Sync** (do not wait for `refreshInterval`)
   ```bash
   kubectl annotate externalsecret llm-api-key-external \
     force-sync=$(date +%s) \
     --overwrite -n openclaw-core
   ```
   Repeat for each affected ExternalSecret in its respective namespace.

4. **Rolling Restart**
   Once the new credential is provisioned in Vault and ESO has synced:
   ```bash
   kubectl rollout restart deployment/openclaw-agent -n openclaw-core
   kubectl rollout restart deployment/whatsapp-connector -n openclaw-integrations
   ```
   Monitor rollout: `kubectl rollout status deployment/openclaw-agent -n openclaw-core`

5. **Audit Trail**
   Record in your incident log:
   - Revocation timestamp
   - Affected pod names: `kubectl get pods -n openclaw-core --no-headers -o custom-columns=":metadata.name"`
   - The new Vault secret version: `vault kv metadata get secret/openclaw/llm`
   - Who performed each step

6. **Canary Verification**
   Before closing the incident, send a test request through the affected integration path and confirm it succeeds with the new credential.

7. **Post-mortem**
   - Update the `secret-expiry-date` annotation on the ExternalSecret to reflect the new schedule
   - Review whether the rotation policy (`secret-rotation-policy`) should be shortened
   - If the compromise was due to a logging leak, audit ConfigMaps to ensure no secrets are stored there

### 6.4 OAuth Token Rotation for Google Workspace

OAuth access tokens expire every hour. Storing them as static K8s Secrets is not viable. The token-refresh sidecar pattern (shown in Section 2.2) handles this:

```
[Vault]
   └── google-oauth-tokens Secret (refresh token only — longer-lived)
         └── token-refresher sidecar reads GOOGLE_REFRESH_TOKEN
               └── sidecar exchanges with Google Token endpoint every 45 min
                     └── writes access_token to /var/tokens/access_token (emptyDir)
                           └── main connector reads access_token from /var/tokens/
```

This ensures:
- The long-lived refresh token is stored securely in Vault (never in env vars)
- The short-lived access token exists only in memory (emptyDir) and is never persisted to a K8s Secret
- Rotation is automatic and continuous without pod restarts

---

## 7. RBAC Roles and RoleBindings

### 7.1 ServiceAccounts

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: openclaw-agent-sa
  namespace: openclaw-core
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: google-workspace-sa
  namespace: openclaw-integrations
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: whatsapp-connector-sa
  namespace: openclaw-integrations
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: discord-connector-sa
  namespace: openclaw-integrations
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eso-controller-sa
  namespace: openclaw-secrets-mgmt
```

### 7.2 Key Security Principle: Agent Must Not Access Integration Credentials

The OpenClaw agent orchestrates tasks and calls integration connectors via gRPC/REST. However, the agent's ServiceAccount (`openclaw-agent-sa`) must **not** have access to secrets in `openclaw-integrations`.

**Why?** If an attacker compromises the agent pod (e.g., via prompt injection), they should not be able to exfiltrate Google OAuth tokens, WhatsApp API tokens, or Discord bot tokens. The integration connectors own their own secrets; the agent only knows the connectors' internal service endpoints.

### 7.3 RBAC Permissions Matrix

| ServiceAccount | Can Read Its Own Secrets | Can Read Other SA's Secrets | Can Write Secrets | Can List All Pods |
|---|---|---|---|---|
| `openclaw-agent-sa` | `llm-api-key` only (`resourceNames`) | No | No | No |
| `google-workspace-sa` | `google-service-account`, `google-oauth-tokens` | No | No | No |
| `whatsapp-connector-sa` | `whatsapp-api-token`, `webhook-signing-secret` | No | No | No |
| `discord-connector-sa` | `discord-bot-token` | No | No | No |
| `eso-controller-sa` | All secrets in managed namespaces | Yes (sync targets) | Yes (writes synced secrets) | No |

### 7.4 Role Definitions

**OpenClaw Agent Role (openclaw-core):**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: openclaw-agent-role
  namespace: openclaw-core
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["llm-api-key"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: openclaw-agent-role-binding
  namespace: openclaw-core
subjects:
  - kind: ServiceAccount
    name: openclaw-agent-sa
    namespace: openclaw-core
roleRef:
  kind: Role
  name: openclaw-agent-role
  apiGroup: rbac.authorization.k8s.io
```

**Google Workspace Connector Role:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: google-workspace-role
  namespace: openclaw-integrations
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["google-service-account", "google-oauth-tokens"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
```

**ESO Controller ClusterRole (CRD watching) + scoped RoleBindings:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: eso-controller-clusterrole
rules:
  - apiGroups: ["external-secrets.io"]
    resources: ["externalsecrets", "secretstores", "clustersecretstores"]
    verbs: ["get", "list", "watch", "update", "patch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
# Bind ESO to each application namespace (not cluster-wide Secret write)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: eso-controller-binding
  namespace: openclaw-core
subjects:
  - kind: ServiceAccount
    name: eso-controller-sa
    namespace: openclaw-secrets-mgmt
roleRef:
  kind: ClusterRole
  name: eso-controller-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

> **RBAC vs NetworkPolicy:** RBAC restricts what a pod can do with the **Kubernetes API** (reading Secrets, ConfigMaps, etc.). It does NOT restrict pod-to-pod network communication. Cross-namespace calls (agent → integrations connector) are governed by NetworkPolicy (see Section 8).

---

## 8. Inter-Service Communication

### 8.1 Topology Diagram

```
External Platforms                Cluster (openclaw-*)
─────────────────────────────────────────────────────────────────
[WhatsApp Business]  ──webhook──→  [WhatsApp Connector]  ─┐
[Discord Gateway]    ←─WebSocket── [Discord Connector]   ─┤
[Google Workspace]   ──push notif→ [Google Connector]    ─┤
                                                           │ gRPC
                                                           ↓
                                         [OpenClaw Agent]
                                                │
                                                │ REST
                                                ↓
                                         [LLM Service]
                                                │
                          ─────────────────────┘
[OpenClaw Agent]  ─gRPC─→ [Integration Connectors]  (to send replies)
```

### 8.2 DNS Service Discovery

| Call | DNS Name | Protocol |
|---|---|---|
| Agent → LLM | `llm-service.openclaw-llm.svc.cluster.local:8080` | REST HTTPS |
| Agent → Google Connector | `google-workspace-service.openclaw-integrations.svc.cluster.local:9090` | gRPC |
| Agent → WhatsApp Connector | `whatsapp-webhook-service.openclaw-integrations.svc.cluster.local:8090` | REST |
| Agent → Discord Connector | `discord-connector-service.openclaw-integrations.svc.cluster.local:8091` | REST |
| Google Connector → Agent | `openclaw-agent-service.openclaw-core.svc.cluster.local:8080` | gRPC |

### 8.3 NetworkPolicy — Cross-Namespace Communication

**Agent egress policy (openclaw-core → integrations and LLM):**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: agent-egress-policy
  namespace: openclaw-core
spec:
  podSelector:
    matchLabels:
      app: openclaw-agent
  policyTypes:
    - Egress
  egress:
    # Allow calls to integration connectors
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: openclaw-integrations
      ports:
        - port: 8090
        - port: 8091
        - port: 9090
    # Allow calls to LLM service
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: openclaw-llm
      ports:
        - port: 8080
    # Allow HTTPS for external LLM API (if ExternalName service)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - port: 443
    # DNS
    - ports:
        - port: 53
          protocol: UDP
```

**Integration connectors egress (openclaw-integrations → agent for routing replies):**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: integrations-egress-policy
  namespace: openclaw-integrations
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: openclaw-core
      ports:
        - port: 8080
    # Allow external API calls (WhatsApp, Discord, Google)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
      ports:
        - port: 443
    - ports:
        - port: 53
          protocol: UDP
```

### 8.4 mTLS Recommendation

For agent-to-connector communication, install **Istio** or **Linkerd** as a service mesh to enable automatic mutual TLS (mTLS) between pods — without code changes.

**Why mTLS here?** The OpenClaw agent sends user email content and private messages to the LLM service. If an attacker gains access to the cluster network (e.g., via a compromised pod), they could intercept this sensitive data in transit. mTLS ensures that only authenticated pods with valid certificates can receive this traffic — preventing both eavesdropping and impersonation attacks within the cluster.

With Istio: annotate namespaces with `istio-injection: enabled` and apply a `PeerAuthentication` policy requiring STRICT mTLS across `openclaw-*` namespaces.

---

## 9. Open Questions / Next Steps

1. **OpenClaw state management**: This plan uses a Deployment (1 replica). If conversational context must persist across pod restarts, evaluate a StatefulSet with PVC or Redis for session storage before going to production.

2. **Discord sharding**: Discord's gateway connection limit means scaling beyond 1 Discord connector replica requires implementing bot sharding. Plan for this if the bot will be in many servers.

3. **Google Workspace OAuth**: The OAuth refresh token in `google-oauth-tokens` secret requires the user to complete an initial OAuth consent flow. Document this one-time setup procedure separately; it cannot be automated by this K8s plan.

4. **Self-hosted LLM GPU node pool**: If choosing Option B (self-hosted LLM), provision a GPU node pool separately before deploying. The `nvidia.com/gpu` resource limit requires the NVIDIA device plugin to be installed on nodes.

5. **Vault setup**: This plan assumes an existing Vault cluster at `vault.internal:8200`. If Vault is not yet available, you can start with native K8s Secrets (with manual rotation) and migrate to ESO later.

6. **Domain and TLS**: Replace `webhook.yourdomain.com` in the Ingress with your actual domain. Install cert-manager and configure the `letsencrypt-prod` ClusterIssuer before applying the Ingress.
