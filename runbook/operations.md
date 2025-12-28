# Operations

## What the platform runs
- End-user app (“Echo”): transcription + diarization + translation + summarization + OCR for video
- Shared APIs: LLM API (Ollama), OCR API, Translation API
- Task queue: Asynq + Redis
- Data stores: Mongo (auth/app data), Vector DB (embeddings/RAG)
- Storage: NFS filesystem + SMB integration + MinIO (S3-compatible)
- JupyterHub: one container per user with persistent storage
- Observability: Prometheus + Grafana + Loki; AsynqMon for queue visibility

## Capacity Guardrails (hard limits)
Reason: single GPU, mixed workloads, prevent contention spiral.
- Max concurrent jobs: 3–4
- Max video length: ~3 hours
- Max file size: ~50 GB
- Artifact retention: short-lived (TTL), only keep what’s required for operational traceability

---

## Data Handling & Retention
- Uploads and intermediate artifacts stored on internal storage (NFS/SMB + MinIO).
- Intermediate artifacts are retained briefly for debugging/traceability, then purged automatically.
- Outputs retained according to internal policy (keep this description generic in public).

---

## GPU Contention Policy
Reality: with one GPU, contention is the default state.

General rules:
- Prefer predictable throughput over “best effort” concurrency.
- Schedule and cap GPU-heavy steps.
- Keep interactive user workflows responsive by prioritizing them over batch-style workloads where possible.

---

## JupyterHub GPU Usage Model
- One container per user, persistent storage enabled.
- GPU access is controlled to reduce contention and avoid VRAM-related instability.
- A dynamic TTL and load-balancing strategy is used to manage notebook workloads competing for GPU resources.

---

## Monitoring Baselines
- GPU memory (alloc/used), OOM counts, and “fragmentation-like” patterns
- Queue depth (pending/active), job latency, retries/failures (AsynqMon helps)
- Disk usage (NFS/MinIO), artifact TTL effectiveness
- API error rates and worker restart counts



