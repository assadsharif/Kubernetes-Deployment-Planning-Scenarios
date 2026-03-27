# Secrets Management Reference

Patterns for managing Kubernetes Secrets with rotation, expiry tracking, and compromise response.

---

## ExternalSecrets Operator (ESO) Pattern

The recommended production approach. K8s Secrets become synced copies of the authoritative credential stored in an external secret store (Vault, AWS SM, GCP SM).

### SecretStore (Vault with Kubernetes auth)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-secret-store
  namespace: <target-namespace>
spec:
  provider:
    vault:
      server: "https://vault.internal:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "<vault-role-name>"
```

### ExternalSecret Template

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: <secret-name>-external
  namespace: <target-namespace>
  annotations:
    secret-expiry-date: "<YYYY-MM-DD>"
    secret-rotation-policy: "<N-days>"
    secret-owner: "<team-name>"
spec:
  refreshInterval: 1h            # How often ESO polls the external store
  secretStoreRef:
    name: vault-secret-store
    kind: SecretStore
  target:
    name: <k8s-secret-name>      # Name of the K8s Secret to create/update
    creationPolicy: Owner
  data:
    - secretKey: <k8s-key>       # Key in the K8s Secret
      remoteRef:
        key: <vault-path>        # Path in Vault (e.g., myapp/credentials)
        property: <vault-field>  # Field within the Vault secret
```

---

## Secret Rotation Flow

```
[External Store: Vault / AWS SM / GCP SM]
     │
     │  1. Platform team updates credential in external store
     │
     ▼
[ESO Controller]
     │
     │  2. ESO detects change on next refreshInterval (≤ 1h)
     │     Or: force immediate sync (see Emergency Sync below)
     │
     ▼
[K8s Secret (synced copy)]
     │
     │  3. Reloader annotation on Deployment detects Secret change
     │     annotation: reloader.stakater.com/auto: "true"
     │
     ▼
[Rolling Update]
     │
     │  4. New pods start with updated credential
     │  5. Old pods drain gracefully (terminationGracePeriodSeconds)
```

---

## Compromised Secret Operational Runbook

Use when a credential is compromised, leaked, or a rotation is overdue.

**Step 1 — Detection**
- Monitoring alert fires (secret age exceeds `secret-expiry-date` annotation, or provider notification)
- Check all expiry annotations:
  ```bash
  kubectl get externalsecret -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}: expiry={.metadata.annotations.secret-expiry-date}{"\n"}{end}'
  ```

**Step 2 — Immediate Revocation**
Revoke the credential at the issuing provider. Do this FIRST — before K8s changes.
- OpenAI / LLM API key: Dashboard → API Keys → Revoke
- Google service account: GCP IAM → Service Accounts → Disable key
- WhatsApp: Meta Developer Portal → Access Tokens → Invalidate
- Discord bot: Discord Developer Portal → Bot → Reset Token
- Vault-managed: `vault lease revoke <lease-id>` or `vault kv delete secret/<path>`

**Step 3 — Force ESO Sync** (skip the refreshInterval wait)
```bash
# Force re-sync by updating the force-sync annotation
kubectl annotate externalsecret <secret-name>-external \
  force-sync=$(date +%s) \
  --overwrite -n <namespace>
```

**Step 4 — Rolling Restart**
Once the new credential is in Vault and ESO has synced:
```bash
kubectl rollout restart deployment/<deployment-name> -n <namespace>
kubectl rollout status deployment/<deployment-name> -n <namespace>
```

**Step 5 — Audit Trail**
Record in your incident log:
- Revocation timestamp
- Affected pods: `kubectl get pods -n <namespace> -o wide`
- New credential version in Vault: `vault kv metadata get secret/<path>`
- Operator who executed each step

**Step 6 — Canary Verification**
Send a smoke-test request through the affected path. Confirm it succeeds with the new credential before declaring the incident resolved.

**Step 7 — Post-mortem**
- Update `secret-expiry-date` annotation on the ExternalSecret
- Shorten `secret-rotation-policy` if the compromise was due to a stale credential
- Audit ConfigMaps to ensure no sensitive values were stored there instead of Secrets

---

## Expiry Annotation Convention

Add these annotations to all ExternalSecret and Secret resources:

```yaml
metadata:
  annotations:
    secret-expiry-date: "YYYY-MM-DD"         # When the credential expires
    secret-rotation-policy: "90-days"        # How often it should be rotated
    secret-owner: "platform-team"            # Who is responsible for rotation
```

Set up a monitoring job (e.g., a CronJob or Prometheus rule) to alert when `secret-expiry-date` is within 14 days of the current date.

---

## OAuth Token-Refresh Sidecar Pattern

OAuth access tokens expire every hour and cannot be stored as static K8s Secrets. Use a sidecar container to continuously exchange the refresh token for a fresh access token.

```yaml
spec:
  containers:
    # Main application container
    - name: main-app
      image: registry/my-connector:latest
      volumeMounts:
        - name: token-cache
          mountPath: /var/tokens          # Reads access_token from here
      # Do NOT put REFRESH_TOKEN here — sidecar handles that

    # Token-refresh sidecar
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
        - name: REFRESH_TOKEN
          valueFrom:
            secretKeyRef:
              name: oauth-tokens-secret    # Only refresh token stored in Secret
              key: REFRESH_TOKEN
        - name: TOKEN_REFRESH_INTERVAL    # Refresh every 45 min (before 1hr expiry)
          value: "2700"
        - name: TOKEN_OUTPUT_PATH
          value: "/var/tokens/access_token"
        - name: TOKEN_ENDPOINT
          value: "https://oauth2.googleapis.com/token"
      volumeMounts:
        - name: token-cache
          mountPath: /var/tokens          # Writes access_token here

  volumes:
    # Shared memory volume: sidecar writes, main app reads
    # emptyDir is ephemeral — access token never persisted to disk or K8s Secret
    - name: token-cache
      emptyDir:
        medium: Memory     # Optional: keep in RAM for extra security
```

**Why emptyDir?**
- The access token exists only in the pod's memory filesystem
- It is never stored in a K8s Secret (which would be persisted to etcd)
- If the pod is terminated, the token is gone — the sidecar will get a fresh one on restart
