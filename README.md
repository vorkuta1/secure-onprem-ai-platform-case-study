# Case Study: Building a Secure On-Prem GPU Inference Platform (Single H100, ~100 Users)

## TL;DR
I designed, deployed, and operated an on-prem AI inference platform under strict data-sovereignty constraints, migrating workflows from CPU-only cloud infrastructure to GPU-enabled on-prem. The platform supports ~100 internal users and processes audio at ~30× real-time (≈ 1 hour of audio in ~2 minutes, workload dependent), while running multiple services concurrently on a single H100 split across DEV/STAGING and PROD environments.

This write-up is NDA-safe: no proprietary code, no internal names, and no sensitive data sources. It focuses on architecture, constraints, operational decisions, and failure modes.

---

## Context
I worked in a public-sector environment with strong compliance requirements around restricted/sensitive data. Prior to this platform:

- We had a cloud bare-metal server without GPU capacity (CPU-only workflows).
- We had a single physical GPU workstation in-office used as a development machine, not suitable as a shared production server.

This created a hard ceiling on:
- **Speed** (CPU-only)
- **Cost efficiency** (CPU time at scale)
- **Confidentiality** (restricted data could not be processed outside the internal network)

---

## Goals
1. Enable GPU-accelerated inference fully **on-prem** (no sensitive data leaving the network).
2. Provide a **reliable self-serve application** for analysts: transcription + diarization + translation + summarization, with OCR for video when needed.
3. Support **multiple internal teams**: analysts using a UI + data scientists using notebooks.
4. Operate sustainably with a small team: platform ownership + simple release process + observability.

---

## Non-Negotiable Constraints
- **Data sovereignty:** restricted/sensitive workloads must stay within the internal network (cloud GPUs were not a viable option even if available).
- **Hardware constraint:** a **single H100** had to support multiple services and user workflows.
- **Operational simplicity:** Kubernetes migration was evaluated but not adopted due to operational cost vs. benefit for the environment.
- **Mixed user base:** non-technical users needed a UI; technical users needed GPU notebooks.

---

## System Overview
The platform is built as a set of composable services (“bricks”) plus one primary end-user application.

### Core components
- **End-user application ("Echo")**
  - Web UI for transcription + diarization + translation + summarization
  - Optional OCR when input is video
  - Auth, admin controls, and basic metrics

- **Task queue + workers**
  - Asynchronous job handling using an Asynq-based queue
  - Worker containers for pipeline steps (ASR, diarization, OCR, translation, summarization)

- **Shared inference services**
  - Central LLM API (Ollama)
  - Standalone OCR / translation APIs
  - Additional specialized pipelines composed of these services + optional standalone models

- **Data science environment**
  - A GPU-enabled **JupyterHub** where data scientists can log in and work inside their own container (terminal + notebooks), with controlled GPU access.

- **Observability**
  - VM-level and application-level monitoring (metrics/logs), sufficient to diagnose performance, queue backlogs, and GPU memory issues.

### Environments & compute layout
- One physical **H100 GPU**
- Split into **two VMs**:
  - **DEV/STAGING VM**
  - **PROD VM**

---

## Deployment Model (CI/CD + Ops)
The release flow was designed to be simple and reliable:

1. Developers push to GitLab.
2. GitLab CI builds container images and publishes them to the GitLab registry.
3. Portainer pulls updated images and updates stacks in the target environment.
4. Separate orchestration:
   - Docker Compose for most services
   - Swarm used for select multi-container stacks requiring it

This provided reproducible deployments without the overhead of a full Kubernetes operational model.

---

## Performance & Scale
- **Users:** ~100 internal users on the main application.
- **Throughput:** ~30× real-time audio processing (≈ 1 hour of audio in ~2 minutes, workload dependent).
- **Operational limits (pragmatic guardrails):**
  - Max video length: ~3 hours
  - Max concurrent jobs: ~3–4
  - Max file size: ~50 GB

These limits were explicitly set to protect system stability under single-GPU constraints.

---

## What Went Wrong (and What I Did About It)
Operating multiple GPU workloads on a single card is mostly an exercise in managing physics and memory, not arguing with Docker.

### 1) OOM and VRAM fragmentation
**Symptom:** jobs failing under load despite “enough” VRAM on paper.  
**Root causes:** fragmentation and uneven allocation patterns across concurrent services.

**Mitigations:**
- Controlled concurrency and job scheduling (hard caps)
- Service-level resource discipline (batch sizing, model loading patterns)
- Operational runbooks for recovery when fragmentation accumulated

### 2) GPU contention across services
**Symptom:** unpredictable latency, queue backlog, and degraded UX when multiple services competed for GPU at once.  
**Reality:** fine-grained GPU partitioning and strict per-service GPU reservations are limited in container orchestration when you only have one physical GPU.

**Mitigations:**
- “Guardrail” concurrency limits per workflow
- Workload prioritization (interactive jobs vs. batch jobs)
- Clear capacity signaling to users (queue/backlog visibility)

**Long-term fix:** more GPUs/nodes to properly allocate and load balance workloads. The clean solution is hardware, not heroics.

---

## Team Model
I acted as the **sole platform/infrastructure engineer**:
- owned architecture, deployment, operations, and reliability
- mentored junior devs and led ~3 data scientists day-to-day
- integrated prototypes produced in notebooks into usable, deployed services

Data scientists:
- developed analysis prototypes in notebooks for internal requests
- validated model behavior on specific tasks

I converted prototype code into production services with authentication, queueing, observability, operational limits, and deploy pipelines.

---

## Why This Matters (Transferable Value)
This project is a strong proxy for real-world production engineering:
- Running GPU inference under constraints (single card, mixed workloads)
- Practical CI/CD and environment separation (DEV/STAGING vs PROD)
- Systems thinking: reliability, capacity planning, guardrails
- Cross-functional delivery: non-technical users + technical research users
- Security/compliance aware architecture (data stays inside the network)

---

## What I’d Do Next
If I were extending this platform:
1. Add hardware capacity (additional GPUs/nodes) to eliminate contention as a class of problems.
2. Introduce explicit scheduling policies (priority queues, preemption where safe).
3. Harden observability around GPU memory behavior and queue dynamics.
4. Standardize a “pipeline SDK” to reduce variance between prototype and production services.

