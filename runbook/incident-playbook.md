# Incident Playbook

## First Response Checklist (Always)
1) Confirm scope: which workflows are impacted (Echo UI, workers, OCR, translation, JupyterHub)?
2) Check queue depth + job latency in AsynqMon.
3) Check GPU memory + OOM/restarts.
4) Check storage saturation (NFS/MinIO) and artifact TTL behavior.
5) Check auth layer (local vs Keycloak/AD).

---

## Incident: GPU OOM / VRAM Fragmentation
### Symptoms
- Jobs fail immediately or mid-run
- Workers restart repeatedly
- Latency spikes without obvious CPU bottlenecks

### Likely causes
- Fragmentation from repeated allocations + mixed workloads
- Too many concurrent GPU workloads
- Model load patterns causing peaks

### Actions
- Reduce concurrency caps (temporary emergency setting)
- Drain/stop non-essential workloads (batch tasks, optional pipelines)
- Restart affected workers/services to reclaim memory
- If safe: stagger restarts to avoid simultaneous model reload storms

### Post-incident hardening
- Keep concurrency caps explicit and enforced at API/queue level
- Introduce “admission control” (reject or defer new jobs when GPU pressure is too high)
- Improve visibility: GPU mem over time + queue depth correlations

---

## Incident: GPU Contention (Slow, not failing)
### Symptoms
- Jobs complete but take much longer
- Queue backlog grows
- Interactive UX becomes sluggish

### Actions
- Prioritize interactive workflows; throttle background/batch
- Temporarily disable non-critical GPU consumers (e.g., heavy notebooks)
- Communicate status: “degraded performance, ETA based on queue depth”

### Structural fix
- Additional GPUs/nodes to allocate per-service GPUs and load balance properly

---

## Incident: Queue Backlog / Stuck Jobs
### Symptoms
- Pending queue grows, active stays flat
- Jobs retry endlessly or appear “stuck”

### Actions
- Check Redis health and worker connectivity
- Check worker logs (Loki) for repeated exceptions
- Pause intake if backlog threatens storage or GPU stability
- Restart a subset of workers (avoid restarting everything at once)

---

## Incident: Storage Saturation (NFS/MinIO)
### Symptoms
- Uploads fail
- Jobs fail on read/write
- System becomes slow or unstable

### Actions
- Identify largest contributors (artifacts, uploads, logs)
- Verify TTL cleanup job is running
- Temporarily reduce max file size or reject new uploads
- Clean up expired artifacts; expand storage only as a last resort if policy allows

---

## Incident: Auth / Keycloak / AD Issues
### Symptoms
- Users cannot login
- Token validation failures
- UI works but API rejects requests

### Actions
- Switch to local auth fallback
- Validate time sync and token expiry behavior
- Restore Keycloak/AD connectivity; test with one user before broad “all clear”

---

## Incident Story: JupyterHub VRAM Contention
### Symptoms
- Data scientist notebooks fail with VRAM errors during peak usage
- Echo jobs and notebooks interfere when running concurrently

### Root cause
- Multiple user containers competing for a single GPU without robust per-container GPU isolation.

### Fix
- Implemented a dynamic TTL and load-balancing strategy:
  - GPU-heavy notebook activity is time-bounded
  - contention is reduced by scheduling/rotating workloads
  - interactive + production workloads are protected via prioritization and concurrency controls

### Result
- Reduced VRAM-related notebook failures and improved platform stability during peak use.

