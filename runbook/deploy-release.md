# Deploy & Release

## Overview
Release pipeline:
1) Push to GitLab
2) GitLab CI builds images and pushes to GitLab Container Registry
3) Portainer pulls new images and updates stacks (DEV/STAGING first, then PROD)

This stack uses Docker Compose for most services and Swarm for select multi-container apps.

---

## Environments
- DEV/STAGING VM: integration testing, load experiments, quick rollback practice
- PROD VM: stable releases, enforced guardrails (limits on concurrency/size)

---

## Release Procedure (Standard)
### 1) Build
- Merge to the release branch/tag
- GitLab CI:
  - build container images
  - run basic checks (lint/tests where available)
  - publish images to GitLab registry with the commit SHA tag

### 2) Deploy to DEV/STAGING
- Update Portainer stack to use the new image tags
- Validate:
  - health endpoints
  - queue intake/processing
  - GPU memory behavior under light load
  - basic end-to-end run on representative media

### 3) Promote to PROD
- Update PROD stacks (via Portainer Agent pull/update)
- Use "small blast radius" first:
  - one workflow first (or one worker type) if possible
  - verify queue depth stays stable and job latency doesn’t spike

---

## Rollback Procedure
Rollback strategy is image-tag based.
- Revert stack to previous known-good image tags
- Restart affected services/workers
- Verify:
  - backlog drains as expected
  - no persistent DB migrations were required (if migrations exist, use forward-only + compatibility window)

---

## Release Guardrails
- Enforce max concurrent jobs (3–4) and file/video size limits at the API layer.
- Ensure artifacts retention policy is active (short TTL) before major releases.
- Confirm monitoring and logging is healthy before and after deploy.

---

## “Kubernetes migration” note (historical)
Kubernetes was evaluated for long-term orchestration, but the operational overhead was not justified in this constrained on-prem environment. The platform prioritizes reliability and simplicity over orchestration complexity.

