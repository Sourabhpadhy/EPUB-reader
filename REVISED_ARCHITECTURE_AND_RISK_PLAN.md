# Revised Architecture and Risk-Mitigation Plan

## Goal
This document updates the original architecture plan for the EPUB-reader project and fixes the main planning issues identified during a design review. The focus is to make the system more realistic, resilient, and maintainable without breaking the original modular structure.

---

## 1. Corrected System Ownership Model

### Core rule
Module 2 (Go backend) owns the canonical business state. Module 3 (Python worker) owns derived data and inference artifacts.

### Ownership boundaries

1. Module 2 owns:
   - series metadata
   - volume metadata
   - chapter state transitions
   - translation status and job queues
   - glossary approval state
   - spoiler guard state
   - websocket subscription state
   - user settings and provider config

2. Module 3 owns:
   - OCR results
   - NLP extracted entities and glossary candidates
   - translation batches and verification output
   - embedding generation
   - vector index and LanceDB tables
   - TTS synthesis
   - export generation files

3. Shared contract rules:
   - Module 2 is the only source of truth for job status and user-facing state.
   - Module 3 returns results with explicit job IDs and verification metadata.
   - Module 3 must never mutate Module 2-owned business state directly.
   - Module 2 must validate all results before marking a chapter complete.

This removes the previous ambiguity where Modules 2 and 3 both appeared to own similar data objects.

---

## 2. Corrected Queue and Worker Model

### Recommended queue layers
Instead of a single monolithic queue, split responsibilities into three layers:

1. Ingestion queue
   - file upload
   - parsing
   - chapter segmentation
   - OCR if needed

2. Translation queue
   - glossary approval
   - chapter translation jobs
   - verification step
   - retry logic

3. RAG / TTS / Export queue
   - vector indexing
   - RAG queries
   - audio generation
   - export packaging

### Fairness rules
- Reader-jump jobs remain high priority.
- Background jobs must age out and move forward using a fairness window.
- No job should be starved forever.
- A chapter will not be translated more than once in parallel.
- Each job must have a unique `job_id`, `series_id`, `volume_id`, and `chapter_num`.

### Queue safety rules
- Every queued job must be idempotent.
- If a retry occurs, the system must detect an already-running job and avoid duplicate work.
- The queue must reject invalid states such as `TRANSLATING -> VERIFIED -> INDEXING` if the preceding step failed.

---

## 3. Resilience and Failure Handling

### Required state machine
Use the following state machine for each chapter:

RAW -> INGESTED -> GLOSSARY_READY -> TRANSLATING -> VERIFIED -> RAG_INDEXED -> READY

Additional failure states:
- FAILED
- RETRYING
- PAUSED
- CANCELLED

### Required behavior
- Module 3 may fail during translation without corrupting the series state.
- If Module 3 crashes after writing partial results, Module 2 must detect inconsistency and rollback or mark the job as failed.
- If a chapter is marked `VERIFIED`, then indexing must happen before `READY`.
- A user question must never use an unverified chapter as RAG context.

### Retry policy
- HTTP 429: backoff and switch to fallback provider
- Timeout: retry up to 2 attempts with exponential backoff
- Worker unavailable: mark job as `RETRYING`, not `READY`
- partial failure: isolate the failed stage and continue other jobs

---

## 4. Health Checks and Connectivity Contracts

### Required health endpoints
Each service must expose:
- `/health/live`
- `/health/ready`
- `/health/worker`

### Required conditions
- Module 3 is considered ready only after:
  - database path is valid
  - required Python packages are installed
  - model load check passes
  - port binding is successful
  - worker can respond to a basic ping request

### Connection rules
- No module should assume localhost-only without fallback config.
- All inter-service calls must support config-driven host and port addresses.
- Module 2 must have connection retries and circuit breakers for Python worker requests.

---

## 5. Bottleneck Reduction and Performance Planning

### Module 2 bottlenecks fixed by design
- Single writer SQLite remains valid, but write operations must be kept minimal and batched.
- Use a dedicated queue worker rather than blocking API handlers with heavy DB writes.
- Keep websocket events per series instead of one giant all-series stream.
- Add rate limits and batching for high-frequency event pushes.

### Module 3 bottlenecks fixed by design
- Do not run all heavy jobs at once.
- Enforce concurrency caps:
  - OCR worker: 1 or 2 jobs
  - glossary NLP: 1 job
  - translation worker: 1 to 2 local jobs depending on GPU memory
  - embedding worker: conservative batch size
  - TTS worker: per user session, not global
- Use model memory budgeting to prevent GPU out-of-memory.
- If compute is overloaded, fall back to cloud or lower-priority lanes.

### Resource safety policy
- No GPU service should run LLM + embeddings + TTS + OCR in the same maximum concurrency burst without explicit safety checks.
- Memory and CPU limits should be enforced with a scheduler.

---

## 6. Better Data Contract Between Modules

### New contract principles
1. Module 2 sends opaque job metadata and full input references.
2. Module 3 responds only with job output, validation metadata, and artifact paths.
3. Module 2 commits final DB state only after it verifies the result.
4. Module 3 never stores business-critical state in a way that bypasses Module 2.

### Required response payload fields
Every worker response should include:
- `job_id`
- `series_id`
- `volume_id`
- `chapter_num`
- `status`
- `verification_passed`
- `error_code` if any
- `artifact_paths` if applicable
- `timestamp`

This makes the pipeline testable and easier to debug.

---

## 7. Frontend / UI Resilience Update

Module 1 should be resilient even when backend services are slow or partially failed.

### UI improvements
- Show job state and queue position clearly.
- Display “translation delayed” or “AI worker offline” messages instead of hanging states.
- Cache the last loaded pages locally and rehydrate after reconnect.
- Retry failed network requests with bounded exponential backoff.
- Support degraded mode for reading even when AI features are temporarily unavailable.

---

## 8. Recommended Final Architecture Fixes

Priority order:
1. Define canonical state ownership and remove ambiguity.
2. Add state machine + retries + health checks.
3. Set concurrency caps and resource limits.
4. Split queues by responsibility.
5. Add per-series websocket routing and better event backpressure.
6. Add observability and queue metrics.
7. Add fallback/degraded mode.

---

## 9. Final Assessment

The original concept is still strong and modular, but the original plan is not yet robust enough for a production-grade version. The corrected plan above fixes the main design flaws without changing the intended architecture.

This revision is still local-first and modular, but it is much more realistic, testable, and maintainable.

---

## 10. Reference
The original architecture remains the foundation. This document is the operational correction layer for performance, stability, and module integrity.

See also:
- `00_Master_Overview_and_Connections.md`
- `module_2/module_2_backend_api_and_database_blueprint.md`
- `module_3/03_AI_NLP_RAG_Pipeline.md`
- `module_1/01_Module_1_Frontend_UI_UX.md`

