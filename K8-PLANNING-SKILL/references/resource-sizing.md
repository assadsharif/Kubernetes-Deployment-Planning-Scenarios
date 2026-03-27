# Resource Sizing Reference

CPU/memory sizing heuristics for common Kubernetes workload archetypes. Use these as starting points — adjust based on load testing.

---

## Sizing Table by Workload Archetype

| Archetype | CPU Request | CPU Limit | Memory Request | Memory Limit | Notes |
|---|---|---|---|---|---|
| **Static frontend** (Next.js, React, nginx) | 50–100m | 200–500m | 64–128Mi | 128–256Mi | File serving; very low compute |
| **REST API (light)** (CRUD, simple logic) | 100–250m | 500m–1 | 128–256Mi | 256–512Mi | DB queries; predictable burst |
| **REST API (heavy / gRPC)** | 250–500m | 1–2 | 256–512Mi | 512Mi–1Gi | Serialization + connection pools |
| **LLM API client / AI agent** | 250–500m | 1–2 | 512Mi–1Gi | 1–4Gi | Context window handling spikes memory |
| **Self-hosted LLM (GPU)** | 4–8 | 8–16 | 8–16Gi | 16–32Gi | GPU workload; dedicated node pool required |
| **Message queue worker** | 100–250m | 500m | 128Mi | 256Mi | I/O bound; predictable |
| **Token-refresh sidecar** | 10m | 50m | 32Mi | 64Mi | Lightweight background loop |
| **Webhook receiver** | 100m | 500m | 128Mi | 256Mi | HTTP fan-out; short-lived per-request |
| **Discord/WebSocket connector** | 100m | 300m | 256Mi | 512Mi | Persistent connection overhead |

---

## Limit-to-Request Ratios

The ratio between limit and request controls burst headroom vs scheduling density trade-off.

| Workload Type | Recommended Ratio | Rationale |
|---|---|---|
| Static frontend | 2:1 to 3:1 | Predictable; some burst for traffic spikes |
| REST API | 2:1 to 3:1 | Occasional burst for complex queries |
| AI agent (LLM client) | 4:1 | LLM context windows cause memory spikes; 4:1 allows burst without reserving it |
| Self-hosted LLM | 2:1 | GPU-bound; memory needs are predictable per model |
| Sidecar processes | 3:1 to 5:1 | Small absolute values; generous ratio is still cheap |

---

## HPA Thresholds by Workload

| Workload | HPA Metric | Target | Min Replicas | Max Replicas |
|---|---|---|---|---|
| REST API | CPU utilization | 70% | 2 | 10 |
| AI agent (LLM client) | CPU utilization | 70% | 2 | 8 |
| Webhook receiver | CPU utilization | 60% | 2 | 20 |
| Self-hosted LLM | Custom (GPU util) | 80% | 1 | 1 (no HPA without model sharding) |

---

## GPU Workload Configuration

For self-hosted LLM pods, add to the pod spec:

```yaml
spec:
  nodeSelector:
    cloud.google.com/gke-accelerator: nvidia-tesla-t4   # Adjust for your cloud/GPU type
  containers:
    - name: llm
      resources:
        requests:
          nvidia.com/gpu: "1"
        limits:
          nvidia.com/gpu: "1"
          cpu: "8000m"
          memory: "16Gi"
```

**Requirements:**
- GPU node pool must exist with the NVIDIA device plugin installed
- `nodeSelector` or `nodeAffinity` must match the GPU node pool labels
- Only one GPU per pod unless model parallelism is implemented

---

## terminationGracePeriodSeconds by Workload

| Workload | Recommended Value | Reason |
|---|---|---|
| Static frontend | 15–30s | Fast drain |
| REST API | 30–60s | Drain in-flight requests |
| AI agent (LLM client) | 120s | LLM inference can run 30–90s |
| AI employee / persistent agent | 180s | Mid-task cleanup |
| WebSocket connector | 30s | Reconnect is fast |
| Self-hosted LLM | 60s | Drain inference requests |
