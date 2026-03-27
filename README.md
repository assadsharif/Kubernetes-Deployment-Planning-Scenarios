# Kubernetes Deployment Planning Scenarios

Two detailed Kubernetes deployment plans and a reusable K8 Planning Skill, built as part of a cloud-native architecture exercise.

## What's Inside

**PLAN-1.md** — Kubernetes deployment plan for an AI Native Task Manager with four components: UI Interface, Backend APIs, Todo Agent (LLM-backed), and Notification Service. Covers namespaces, Deployments with HPA, Services, ConfigMaps, Secrets rotation via ExternalSecrets Operator, RBAC with least-privilege scoping, and NetworkPolicy.

**PLAN-2.md** — Kubernetes deployment plan for an AI Personal Employee (OpenClaw) integrated with Google Workspace, WhatsApp, and Discord. Security-focused design with OAuth token-refresh sidecar, a compromised-secret operational runbook, and mTLS recommendation for sensitive data in transit.

**K8-PLANNING-SKILL/** — A reusable agent skill that generates Kubernetes deployment plans for any future project. Ask it about your system architecture and it produces a structured plan covering all eight K8s deployment concerns.

## Author

Asad Sharif
