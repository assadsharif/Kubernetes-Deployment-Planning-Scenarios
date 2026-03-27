---
name: k8s-deployment-planner
description: >
  Generate comprehensive Kubernetes deployment plans for any project.
  Use when a user wants to plan, design, or document how their application
  should be deployed to Kubernetes — covering namespace design, Deployments,
  Services, ConfigMaps, Secrets management, RBAC, and inter-service
  communication. Trigger on: "k8s plan", "kubernetes deployment design",
  "plan my kubernetes setup", "how do I deploy X to k8s", "write a kubernetes
  plan", "k8s architecture", or any time a user describes a multi-component
  system and asks for a deployment strategy. Also use when the user shows
  an architecture diagram and wants a deployment plan.
type: guide
allowed-tools: Read, Glob, Grep, AskUserQuestion
---

# K8s Deployment Planner

This skill generates structured, production-oriented Kubernetes deployment plans for any multi-component project. It encodes Kubernetes planning expertise so you produce consistent, well-reasoned plans without having to rediscover the same patterns each time.

**Output:** A Markdown document covering all 9 sections below, with annotated YAML snippets, decision rationale tables, and an operational runbook for secrets management.

---

## Phase 1: Clarifying Questions

Before generating the plan, collect the following information. Use `AskUserQuestion` if any of these are not already present in context. Present all questions at once — do not ask them one at a time.

**Questions to ask:**

1. **Component inventory**: List all components — services, AI agents, databases, background workers, external integrations. What does each one do in one sentence?

2. **Communication patterns**: Which components talk to which? What protocol does each connection use? (REST, gRPC, WebSocket, message queue, event bus)

3. **Statefulness**: Which components have persistent state that must survive pod restarts? (Databases, agent session memory, file storage, cache)

4. **External access**: Which components need to be reachable from outside the cluster? (User-facing UIs, webhook receivers, public APIs)

5. **Security sensitivity**: Does the system handle credentials, PII, OAuth tokens, or private user data (emails, messages, files)? What is the blast radius if a credential is compromised?

6. **AI/LLM workloads**: Are any components making LLM API calls (external API) or running a local model (self-hosted)? This drives resource sizing dramatically.

7. **Kubernetes maturity**: Is this a fresh cluster, or extending an existing one? Are there existing ingress controllers, cert-manager, or service mesh installations?

8. **Secret management infrastructure**: Which of these does your team have available?
   - HashiCorp Vault
   - AWS Secrets Manager
   - GCP Secret Manager
   - Azure Key Vault
   - Native K8s Secrets only (manual rotation)

After collecting answers, **confirm your understanding** before generating:

> "Based on your answers, here is what I understand: [summary of components, communication, security needs]. Does this look right before I generate the plan?"

---

## Phase 2: Plan Generation Rules

Apply these rules when generating the plan. Each decision should be explained with a brief rationale in the plan document.

### Namespace Design

- One namespace per logical tier: UI, API, AI/agent, data, integrations, infrastructure
- **AI/LLM workloads always get their own namespace** — their memory footprints are unpredictable and can starve co-located API tiers
- External secret management infrastructure (ESO, Vault agent injector) gets its own namespace — ESO controller needs elevated permissions; isolating it limits blast radius
- Apply `ResourceQuota` to every namespace
  - AI namespaces: higher memory ceiling, allow CPU burst
  - Web/API namespaces: tighter ceiling, predictable workloads
- Label all namespaces with `app.kubernetes.io/part-of`, `tier`, `environment`
- Naming convention: `<project>-<tier>` (e.g., `taskmanager-agents`, `openclaw-integrations`)

See `references/namespace-patterns.md` for ResourceQuota and LimitRange templates.

### Deployment vs StatefulSet Decision Tree

```
Does the component write to local disk state that must survive pod restarts?
├── No → Deployment
│   └── Does it have external state? (database, Redis, external API)
│       ├── Yes → Deployment (stateless per-request, scale horizontally)
│       └── No → Deployment + add external state store to the plan
└── Yes → Does restart ORDER matter across pods?
    ├── Yes → StatefulSet (databases, distributed systems requiring ordered init)
    └── No → Consider Deployment + PVC (simpler) or StatefulSet

AI agents with conversational context:
→ Prefer Deployment + external state (Redis/DB) for easier scaling
→ Use StatefulSet only if context must remain local (e.g., large model weights on disk)
```

### Service Type Selection Rules

| Scenario | Service Type | Note |
|---|---|---|
| User-facing UI or webhook receiver | LoadBalancer + Ingress | Use Ingress for TLS termination and path routing |
| Internal service (API, agent, connector) | ClusterIP | Always default to ClusterIP |
| NodePort | Never in production | Port management is brittle across multi-namespace setups |
| External SaaS API endpoint | ExternalName | Routes cluster DNS to external hostname |

### Resource Sizing

Use the sizing table in `references/resource-sizing.md` to select initial request/limit values by workload archetype.

**Key ratios:**
- API tier: 2:1 to 3:1 limit-to-request (predictable burst)
- AI agent (LLM API client): 4:1 limit-to-request (LLM context windows cause memory spikes)
- Self-hosted LLM: single replica with GPU node selector; no HPA unless model sharding is implemented
- Token-refresh sidecar: minimal resources (10m CPU, 32Mi memory)

Always include `readinessProbe` and `livenessProbe`. Add `startupProbe` for components with slow initialization (gRPC servers, model loading).

For AI agent deployments that handle long-running LLM requests, set `terminationGracePeriodSeconds` to 120s or longer.

Include HPA for AI agent tiers that scale with traffic (autoscaling/v2, CPU target 70%).

### Secrets Management Decision Tree

```
Does the team have an external secret store (Vault, AWS SM, GCP SM)?
├── Yes → Use ExternalSecrets Operator (ESO) pattern
│   ├── SecretStore (points to the external store with K8s auth)
│   ├── ExternalSecret (pulls credential into K8s Secret, sets refreshInterval)
│   ├── Annotate with secret-expiry-date and secret-rotation-policy
│   └── See references/secrets-management.md for full YAML templates
└── No → Native K8s Secrets
    ├── Annotate with secret-expiry-date and secret-rotation-policy
    ├── Document manual rotation procedure
    └── See references/secrets-management.md for rotation runbook

Does the system have OAuth tokens (e.g., Google Workspace integration)?
└── Yes → Token-refresh sidecar pattern
    ├── Store ONLY the refresh token in a K8s Secret
    ├── Sidecar exchanges refresh token for access token every 45 minutes
    ├── Access token written to emptyDir volume (never persisted to K8s Secret)
    └── See references/secrets-management.md for two-container pod spec template
```

**Always include a compromised-secret runbook** in the plan for any system that handles sensitive user data. See `references/secrets-management.md`.

### RBAC Minimum Principles

1. **One ServiceAccount per Deployment** — never share ServiceAccounts across Deployments
2. **Use `resourceNames` to scope Secret access** — `get` on `secrets` without `resourceNames` lets a pod read ALL secrets in the namespace
3. **Least privilege per component** — an AI agent should not be able to read integration credentials; connectors own their own secrets
4. **Cross-namespace network calls = NetworkPolicy, not RBAC** — RBAC controls Kubernetes API access; NetworkPolicy controls pod-to-pod traffic
5. **ESO controller gets a ClusterRole** (needs to watch CRDs cluster-wide) but bound via namespace-scoped RoleBindings to limit write access

See `references/rbac-patterns.md` for pre-built Role and ServiceAccount templates.

---

## Phase 3: Output Template

Always generate the plan using this exact 9-section structure:

```markdown
# Kubernetes Deployment Plan: [Project Name]

**Date:** [today]
**Version:** 1.0

## Architecture Overview
[1-2 paragraphs: what the system does, the key communication patterns,
and what the dominant design driver is (performance, security, scaling, etc.)]

## 1. Namespace Design
[Namespaces table + rationale + Namespace YAMLs + ResourceQuota YAMLs]

## 2. Pods / Deployments / StatefulSets
[One subsection per component: Deployment/StatefulSet YAML + replica rationale]
[Include HPA for AI agent tiers]

## 3. Services and Service Types
[Services table + Service YAMLs + Ingress YAML if applicable]

## 4. Resource Requests and Limits
[Sizing table for all components + rationale for AI/LLM sizing decisions]

## 5. ConfigMaps
[One ConfigMap per namespace + full YAML for AI/agent config]

## 6. Secrets Management
[ConfigMap vs Secret decision matrix]
[Secret YAMLs with expiry annotations]
[ESO pattern if applicable: SecretStore + ExternalSecret YAMLs]
[Secret rotation flow]
[Compromised secret runbook — mandatory if system handles sensitive data]

## 7. RBAC Roles and RoleBindings
[ServiceAccount YAMLs]
[Role YAMLs with resourceNames restriction on all Secret access]
[RoleBinding YAMLs]
[RBAC permissions matrix table]
[Note: cross-namespace traffic = NetworkPolicy, not RBAC]

## 8. Inter-Service Communication
[Communication map table with DNS names and auth methods]
[NetworkPolicy YAMLs for each namespace]
[mTLS recommendation if sensitive data is transmitted between pods]

## 9. Open Questions / Next Steps
[List all assumptions made during plan generation]
[Flag infrastructure prerequisites (Vault, GPU nodes, cert-manager, etc.)]
[Identify one-time setup steps that cannot be automated by K8s manifests]
```

**Section 9 is mandatory.** Every plan makes assumptions. Listing them explicitly prevents teams from deploying a plan against wrong infrastructure expectations.

---

## Reference Files

Load a reference file when its topic is relevant to the project being planned. Do not load all four for simple two-component systems.

| Reference | When to Load |
|---|---|
| `references/namespace-patterns.md` | Always — needed for ResourceQuota templates |
| `references/resource-sizing.md` | When sizing AI/LLM workloads or unfamiliar workload types |
| `references/secrets-management.md` | When system handles credentials, OAuth tokens, or sensitive user data |
| `references/rbac-patterns.md` | When designing ServiceAccount and Role structures |
