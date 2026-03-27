# Namespace Patterns

Reference patterns for Kubernetes namespace design, labeling, and resource governance.

---

## Standard Label Schema

Apply these labels to every namespace for consistent filtering and tooling (Argo CD, Kyverno, monitoring):

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <project>-<tier>
  labels:
    app.kubernetes.io/part-of: <project-name>   # Groups all namespaces for a project
    tier: <tier>                                  # frontend | api | ai-compute | data | integrations | infrastructure
    environment: <env>                            # production | staging | development
```

### Tier Values

| Tier | Use For |
|---|---|
| `frontend` | Web UIs, static file servers |
| `api` | REST and gRPC application backends |
| `ai-compute` | LLM agents, model inference, AI workloads |
| `data` | Databases, caches, message queues |
| `integrations` | Connectors to external SaaS (Google, Slack, etc.) |
| `infrastructure` | ESO, Vault agent injector, cert-manager, monitoring |

---

## Naming Convention

Format: `<project>-<tier>`

Examples:
- `taskmanager-ui`
- `taskmanager-agents`
- `openclaw-integrations`
- `openclaw-secrets-mgmt`

Avoid:
- Single namespace for all components (`my-app`) — no isolation between tiers
- Tier-only names (`agents`, `api`) — not unique across projects in the same cluster

---

## ResourceQuota Templates

### Web / Frontend Tier (tight, predictable)

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: frontend-quota
  namespace: <project>-ui
spec:
  hard:
    requests.cpu: "500m"
    requests.memory: "512Mi"
    limits.cpu: "2"
    limits.memory: "1Gi"
    pods: "10"
    services: "5"
```

### API / Backend Tier (moderate)

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: api-quota
  namespace: <project>-api
spec:
  hard:
    requests.cpu: "2"
    requests.memory: "2Gi"
    limits.cpu: "8"
    limits.memory: "8Gi"
    pods: "20"
    services: "10"
```

### AI / Agent Tier (high memory ceiling, CPU burst)

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ai-quota
  namespace: <project>-agents
spec:
  hard:
    requests.cpu: "2"
    requests.memory: "4Gi"
    limits.cpu: "16"
    limits.memory: "32Gi"   # High ceiling: LLM context windows spike memory
    pods: "20"
    services: "10"
```

### Notifications / Lightweight Services

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: notifications-quota
  namespace: <project>-notifications
spec:
  hard:
    requests.cpu: "500m"
    requests.memory: "512Mi"
    limits.cpu: "1"
    limits.memory: "1Gi"
    pods: "10"
```

---

## LimitRange Template

Applies default resource requests/limits to containers that don't specify their own. Prevents pods from consuming unbounded resources if a developer forgets to set `resources:`.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: <project>-<tier>
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "4"
        memory: "8Gi"
      min:
        cpu: "10m"
        memory: "32Mi"
```

Deploy a LimitRange in every namespace. Without it, a container with no `resources:` block gets no requests/limits — the scheduler can't place it intelligently and a single runaway container can consume all node memory.
