# RBAC Patterns Reference

Templates and guidance for Kubernetes Role-Based Access Control in multi-component, multi-namespace applications.

---

## Core Principle: One ServiceAccount per Deployment

Every Deployment gets its own ServiceAccount. Never share ServiceAccounts across Deployments.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: <component>-sa
  namespace: <component-namespace>
  labels:
    app.kubernetes.io/part-of: <project>
    component: <component>
```

**Why one SA per Deployment?**
- Limits blast radius: if one SA's token is compromised, only that component's permissions are affected
- Enables per-component audit logging (API server logs include SA name)
- Allows per-component network identity (Istio, Linkerd use SA for mTLS cert identity)

---

## Read-Only ConfigMap Role

Most application pods only need to read their own ConfigMap. Grant nothing more.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader-role
  namespace: <namespace>
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: configmap-reader-binding
  namespace: <namespace>
subjects:
  - kind: ServiceAccount
    name: <component>-sa
    namespace: <namespace>
roleRef:
  kind: Role
  name: configmap-reader-role
  apiGroup: rbac.authorization.k8s.io
```

---

## Scoped Secret Access with resourceNames

**Always use `resourceNames` when granting Secret access.** Without it, a pod can read ALL secrets in the namespace — a significant exfiltration risk.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: <component>-secret-reader
  namespace: <namespace>
rules:
  # ConfigMaps: allow listing (no resourceNames restriction needed — low risk)
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]

  # Secrets: restrict to ONLY the exact secret this component needs
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["<specific-secret-name>"]   # ← REQUIRED: limits to one secret
    verbs: ["get"]
```

**What NOT to write:**
```yaml
# DANGEROUS: allows reading ALL secrets in the namespace
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]   # No resourceNames = unrestricted
```

---

## Full Component Role + RoleBinding Template

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: <component>-role
  namespace: <namespace>
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["<secret-1>", "<secret-2>"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: <component>-role-binding
  namespace: <namespace>
subjects:
  - kind: ServiceAccount
    name: <component>-sa
    namespace: <namespace>
roleRef:
  kind: Role
  name: <component>-role
  apiGroup: rbac.authorization.k8s.io
```

---

## ESO Controller ClusterRole Pattern

The ExternalSecrets Operator controller needs cluster-wide permission to watch `ExternalSecret` CRDs, but its Secret write access should be scoped to specific namespaces via RoleBindings — not a ClusterRoleBinding.

```yaml
# ClusterRole: watch CRDs cluster-wide + generic Secret operations
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: eso-controller-clusterrole
rules:
  - apiGroups: ["external-secrets.io"]
    resources: ["externalsecrets", "secretstores", "clustersecretstores",
                "clusterexternalsecrets", "pushsecrets"]
    verbs: ["get", "list", "watch", "update", "patch"]
  - apiGroups: [""]
    resources: ["secrets", "serviceaccounts"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "patch"]
---
# ClusterRoleBinding: only for CRD watching (non-sensitive)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: eso-crd-watcher-binding
subjects:
  - kind: ServiceAccount
    name: eso-controller-sa
    namespace: <project>-secrets-mgmt
roleRef:
  kind: ClusterRole
  name: eso-controller-clusterrole
  apiGroup: rbac.authorization.k8s.io
---
# Namespace-scoped RoleBinding: bind ESO to each app namespace separately
# Repeat this for every namespace where ESO should sync secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: eso-controller-binding
  namespace: <target-app-namespace>
subjects:
  - kind: ServiceAccount
    name: eso-controller-sa
    namespace: <project>-secrets-mgmt
roleRef:
  kind: ClusterRole
  name: eso-controller-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

---

## RBAC vs NetworkPolicy Clarification

> **RBAC restricts access to the Kubernetes API server.** It controls what a pod can DO with K8s resources: read Secrets, create Deployments, list Pods, etc.

> **RBAC does NOT restrict pod-to-pod network communication.** A pod's ServiceAccount having no cross-namespace permissions does not prevent that pod from making HTTP/gRPC calls to services in other namespaces.

Use **NetworkPolicy** for cross-namespace traffic control. Use **RBAC** for Kubernetes API resource access control.

---

## NetworkPolicy Template: Allow Egress to Specific Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-<target-tier>
  namespace: <source-namespace>
spec:
  podSelector:
    matchLabels:
      app: <source-app>         # Scope to specific pods, not all pods in namespace
  policyTypes:
    - Egress
  egress:
    # Allow traffic to the target namespace on specific ports
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: <target-namespace>
      ports:
        - port: <port-number>
          protocol: TCP
    # Always allow DNS
    - ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
```

**Default-deny pattern:** Deploy a default-deny NetworkPolicy in every namespace, then explicitly allow required communication. This ensures unintended cross-namespace calls fail closed.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: <namespace>
spec:
  podSelector: {}       # Applies to ALL pods in namespace
  policyTypes:
    - Ingress
    - Egress
  # No egress or ingress rules = deny all
```
