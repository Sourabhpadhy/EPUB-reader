# Revised Architecture and Risk-Mitigation Plan

**Project:** NovelHub · **Version:** 2.0 (aligned with `00_Master_Overview_and_Connections.md` v3.0) · **Date:** 2026-11-03

This file is the operational correction layer: who owns what, how work survives failure, how resources are protected, and what we test before each release. It adds detail to the master overview and never contradicts it. If the two disagree, the master overview wins.

---

## 1. Ownership model

The Go hub and the separate Python worker service no longer exist. There is one application process and on-demand worker pools.

| Owner | Owns | Rules |
|---|---|---|
| **App process** (FastAPI, one worker) | All business state: series, volumes, chapters, translations, glossary, notes, threads, jobs, settings, spoiler guard; the SSE hub; lane routing; the only SQLite writer | Validates every worker result before committing it. Commits in one transaction. Decides what is "verified" |
| **Worker pools** (child processes, started on demand) | Derived data: parsed structure, OCR text, NLP candidates, embeddings and LanceDB tables, TTS audio, export and backup files | Pure functions with typed inputs and outputs. Never write SQLite. Never decide business state. LanceDB is written by the RAG pool only |
| **LLM endpoints** (local server, cloud lanes) | Inference only | Stateless. Chosen by the routing policy and pinned per chapter |

This removes the old ambiguity where two services appeared to own the same objects.

---

## 2. Jobs and state machines

### 2.1 One durable queue instead of three

A single `jobs` table replaces the three separate queues. Fairness and isolation come from priority tiers and per-kind concurrency caps (§5), not from separate queue implementations.

* **Tiers:** 0 = interactive (not queued), 1 = reader jump, 2 = background. A reader jump promotes one chapter and the next, so background work resumes as soon as those finish. If measurements show background starvation, add aging (a Tier 2 job waiting longer than a set time is treated as Tier 1).
* **Idempotency:** duplicate `(kind, volume, chapter)` jobs are merged. Batch commits are idempotent per `(job_id, batch_idx)`, so a retry never duplicates units.
* **One chapter at a time:** a chapter has at most one active translation job.
* **Retry:** a failed attempt returns the job to `queued` with `attempts + 1` and a `not_before` backoff time. Jobs whose `not_before` is in the future are skipped by the claimer.
* **Pause:** pause = cancel the queued jobs of a volume. Drafts stay; resuming re-enqueues from the first missing batch.

### 2.2 Chapter state machine

```
RAW → QUEUED → DRAFTING → VERIFIED → INDEXED
            ↘        ↘         ↘
              FAILED (with error_code; retry re-enters QUEUED)
```

* Transitions go through one function that rejects illegal moves (for example `INDEXED` without `VERIFIED`).
* A chapter is **readable** from `VERIFIED`. It becomes **searchable by AI** only at `INDEXED`.
* If indexing fails, the chapter stays `VERIFIED` and indexing is retried. Chat answers only from indexed chapters and tells you which chapters are still pending.
* Volume status: `RAW → GLOSSARY_SCANNING → GLOSSARY_REVIEW → TRANSLATING → COMPLETED | FAILED`.

### 2.3 Recovery rules

| Situation | Response |
|---|---|
| App restarts | Every `running` job returns to `queued` |
| Job with a stale `heartbeat_at` (hung worker) | Treated as failed and requeued with backoff |
| Chapter stuck in `DRAFTING` with no active job | Re-enqueued; resumes from the first missing batch |
| Pinned lane must change (rate limit exhausted) | Drafts of that chapter are discarded and it restarts on the new lane, so one chapter never mixes models |

---

## 3. Failure handling

| Failure | Detection | Response |
|---|---|---|
| HTTP 429 from a cloud lane | Status code, `Retry-After` header | Put the lane on cooldown (honour `Retry-After`, default 45 s); retry the batch up to the limit; then restart the chapter on another lane if allowed |
| Timeout | Request timeout | Up to 2 retries with exponential backoff, then the job fails with an `error_code` |
| Local LLM endpoint down | Connection error, `GET /system/status` | Jobs wait in `queued` with `not_before`; UI shows "Local model offline"; reading keeps working |
| Worker process crash (bad PDF/EPUB, out of memory) | `BrokenProcessPool` | The job fails with `worker_crashed`; the pool is recreated; the API stays up. Deterministic parse failures are not retried |
| Malformed model output | Audit (coverage, residual CJK) | Critic repair, maximum 2 attempts, then the unit is `flagged` for review |
| Partial results | Batch commits are drafts | Drafts are promoted to `verified` only after the whole chapter passes; nothing half-visible |
| SQLite write failure | Exception on the single writer | The job fails and retries; no concurrent writers means no lock contention |
| Disk nearly full | Free-space check before ingest, export, backup, restore | Fail early with a clear message |
| Bad backup archive | Manifest and schema validation, staging directory | Reject before touching the live database |
| Failed migration | Automatic database copy taken before migrating | Restore the copy and refuse to start with a clear error |

---

## 4. Health and observability

| Mechanism | Purpose |
|---|---|
| `GET /health` | Liveness: the process is up |
| `GET /system/status` | Readiness detail: schema migrated, data directory writable, free disk, local endpoint reachable and which model, lane status and cooldowns, installed extras, worker pool states, queue depth |
| `novelhub doctor` | The same checks from the terminal, with fix instructions |
| Structured logs | JSON lines with `job_id`, `series_id`, `chapter`, `lane`, duration, and `error_code`. Logs stay on your machine |
| `audit_stats` | Failure rate per audit rule, so the audit itself can be tuned |
| Jobs panel in the UI | Queue position, state, and estimated time per job |

No telemetry leaves the machine.

---

## 5. Resource and concurrency policy

| Work | Cap | Notes |
|---|---|---|
| Local translation | 1 job (2 only if you enable it) | llama-server runs 2 slots; one is reserved for interactive use |
| Cloud lane | Per-lane `concurrency` from lane settings | Rate limits come from settings, not code |
| Ingest and OCR pool | 1 worker (maximum 2) | Pages with little selectable text only |
| Glossary NLP scan | 1 | |
| RAG pool (embed, rerank) | 1 worker, conservative batch size | Models loaded lazily, exit after idle |
| TTS pool | 1 worker; client prefetches 3 units | Kokoro loaded lazily |
| Export and backup | 1 | |

Protective rules:

* **GPU:** only the local LLM server uses the GPU by default. Embeddings, reranking, OCR and TTS run on CPU, so they cannot cause VRAM overflow.
* **RAM gate:** before starting a heavy job the claimer checks free memory against that kind's measured estimate (numbers from Phase 0) and defers the job if it would not fit.
* **The 50 ms rule:** anything expected to use more than about 50 ms of CPU runs in a pool. A loop-lag monitor logs a warning when the event loop stalls.
* **Event backpressure:** SSE hint events are coalesced (a few per second per stream). Dropping a hint is safe because the UI re-fetches state over REST.

---

## 6. Contract between the app and the workers

1. The app passes IDs, file paths and small data slices. Workers read SQLite read-only when they need more.
2. Every worker returns a typed result (Pydantic): `ok`, `error_code`, `payload`, `artifact_paths`, `duration_ms`, `worker_version`.
3. The app validates before committing: schema, sentence coverage, size limits, and that every `artifact_path` is inside the data directory.
4. Outputs record the versions that produced them: `parser_version`, `pagination_version`, `style_guide_rev`.
5. Workers never store business-critical state anywhere except in their returned result or their own derived stores (LanceDB, caches).

HTTP-facing payloads live in `contracts/openapi.json` and `contracts/events.schema.json`; worker result models live in `backend/novelhub/workers/models.py`.

---

## 7. Frontend resilience

* **Events:** SSE with automatic reconnect and `Last-Event-ID`. On reconnect or window focus, the UI re-fetches state over REST. Events are hints only.
* **Requests:** TanStack Query retries failed calls with bounded exponential backoff.
* **Status messages instead of hanging states:** "Local model offline", "Translation delayed, rate limited, retrying in 40 s", "AI unavailable, reading still works".
* **Degraded reading:** the reader works on raw text when translation or AI is unavailable; already loaded pages stay in the query cache.
* **Job visibility:** chapter chips show state and queue position; the jobs panel shows details.
* **Session token:** the token changes when the app restarts. On a 401 the UI reloads once to pick up the new token and shows a short "reconnecting" notice.
* **Optimistic updates** only for progress and notes, with rollback on error.

---

## 8. Risk register

Likelihood and impact are my estimates; revisit them after Phase 0.

| Risk | Likelihood | Impact | Mitigation | How we check |
|---|---|---|---|---|
| Scope too large for one developer | High | High | Phased plan; every phase is usable; open source invites help | Phase exit criteria |
| Spoiler leak through any context path | Medium | High | One `guard_spec` function applied everywhere, including notes and recaps | Property test + 20 adversarial prompts = 0 leaks |
| Local translation quality below expectation | Medium | High | Phase 0 bake-off, cloud lanes, Critic, corrections memory | Blind side-by-side on 3 chapters |
| VRAM or RAM exhaustion | Medium | High | Measured budget, lazy pools, RAM gate, reserved interactive slot | Phase 0 benchmarks; `nvidia-smi` under load |
| Crash mid-translation | Medium | Medium | Draft commits, startup recovery, idempotent batches | Kill-and-resume test |
| Event-loop stalls (heavy work leaks into the app) | Medium | Medium | The 50 ms rule, loop-lag monitor | Loop-lag test while OCR runs |
| Parser variance across EPUB generators | High | Medium | Golden tests on varied fixtures, block-model normalisation | Golden-file suite |
| Security: localhost abuse, SSRF, malicious files | Medium | High | Token plus Host/Origin checks, SSRF-safe fetcher, zip limits, HTML sanitised to blocks | Security checklist per release |
| Database migration loses data | Low | High | Automatic backup before migration, forward-only numbered migrations | Upgrade test from the previous version |
| Dependency license trap (for example AGPL PDF library) | Medium | Medium | License audit in CI; prefer permissive libraries | CI license check |
| Legal or copyright problems in a public repo | Medium | High | No bundled content, public-domain fixtures, disclaimer, no scraping in core | Release checklist |
| Cloud limits or terms change | Medium | Medium | Limits in lane settings, cloud off by default, per-series opt-in | Manual review per release |
| Install friction for other users | High | Medium | Optional extras, `novelhub doctor`, first-run wizard, bring-your-own endpoint | Fresh-machine install test |
| Support load from other platforms | Medium | Medium | Linux first, CI with a mock LLM on all three OSes, issue templates | CI matrix |

---

## 9. Resilience test plan (run before every release)

| Scenario | Expected result |
|---|---|
| `kill -9` the app mid-chapter | Restart resumes from the first missing batch; no duplicate or lost units |
| Stop the local LLM endpoint mid-batch | Job returns to `queued`; UI shows the banner; no corrupted drafts |
| Simulate a 429 storm on a cloud lane | Cooldown applied; chapter restarts on another lane; never mixes models |
| Feed a corrupt or malicious EPUB/PDF | The job fails cleanly; the API stays responsive; no file escapes the data directory |
| Run OCR or embedding while scrolling the reader | Event-loop lag stays under the threshold set in Phase 0 |
| Restore a backup on a fresh data directory | Identical IDs and rendered text |
| Upgrade from the previous release | Automatic backup taken; migration succeeds; rollback works |
| Move the guard backward | Chat, recap, character cards and notes context shrink immediately |

---

## 10. When each fix lands

| Phase | Resilience work |
|---|---|
| 0 | Ownership rules, worker result models, `/health` and `/system/status`, `novelhub doctor` stub, CI with the mock LLM, license audit |
| 1 | SSE client with reconnect, status banners, session-token handling, free-space checks, migration backups |
| 2 | Chapter state machine, job recovery, idempotent batch commits, concurrency caps, RAM gate, audit statistics |
| 3 | Lane cooldowns, per-chapter pinning, restart-on-lane-change |
| 4 | Guard property and adversarial tests, pending-index handling in chat |
| 5 | Backup and restore tests, export failure handling |
| 7 | Fresh-machine install test, release checklist, full resilience test plan |

---

## 11. What changed from the previous version of this document

| Previous version | Now |
|---|---|
| Module 2 (Go) owns state, Module 3 (Python) owns derived data | One app process owns state; worker pools own derived data |
| Three queue layers (ingest, translation, RAG/TTS/export) | One `jobs` table with tiers and per-kind concurrency caps |
| State machine with `READY`, `RETRYING`, `PAUSED`, `CANCELLED` on chapters | Chapter states stay simple; retry, pause and cancel live on jobs |
| Circuit breakers for Python worker requests | Not needed in-process; worker crashes are handled by pool recreation |
| Health endpoints `/health/live`, `/health/ready`, `/health/worker` | `/health`, `/system/status`, `novelhub doctor` |
| Per-series WebSocket routing and event batching | One SSE stream of small hint events; the UI re-fetches over REST |
| Worker responses carry `job_id`, `series_id`, `volume_id` | Workers return typed results; the app attaches job context and validates before commit |
| No release or data-safety rules | Migration backups, resilience test plan, open-source release checks |

---

## 12. References

* `00_Master_Overview_and_Connections.md`: architecture, data model, phases, open-source plan
* `FEATURES_AND_IDEAS.md`: feature behaviour, deferred items, ideas
