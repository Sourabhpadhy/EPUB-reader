# 03_Module_3_AI_NLP_RAG_TTS.md
**Project Name:** NovelHub — Local-First Series ePub/PDF/Web Novel Reader, Translator & RAG Workspace  
**Module:** Module 3 — Python AI/ML, CJK NLP, Hybrid Translation, Post-Verify RAG & Pluggable TTS Worker (`:8001`)  
**Target Environment:** CachyOS Linux (Hyprland) | Intel Core i7-13th Gen | 16 GB System RAM | NVIDIA RTX 4060 (8 GB VRAM)  
**Upstream Contract:** Strictly adheres to `00_Master_Overview_and_Connections.md` and `02_Module_2_Backend_API_Database.md` (Bridge B `/internal/*` on `:8001`)

---

## 1. Execution Priority & Hardware Resource Split

### Core Architectural Rule: Translation-First + Post-Verification RAG Gate
When a volume is imported, compute focuses on parsing and translation first. RAG vector indexing never runs on raw untranslated text; instead, Go (`Module 02`) triggers `/internal/rag/index-chapter` **only after** a chapter passes Stage 4 verification (`status = 'VERIFIED'`). Furthermore, to prevent SQLite write-lock contention with Go's single-writer pool, **Python opens `data/novelhub.db` in Read-Only mode (`?mode=ro`)** and returns all mutated records in the HTTP JSON response for Go to commit.

| Subsystem | Model / Library | Assigned Hardware | Memory Budget | Execution Timing |
| :--- | :--- | :--- | :--- | :--- |
| **Local Translation & Chat SLM** | `Qwen3-8B` (`Q4_K_M`, `/no_think` for translation, `/think` for deep review) or `Sakura-7B-Qwen2.5` via `llama-server :8081` | **RTX 4060 GPU (100% Dedicated)** | **~6.37 GB VRAM** (Weights + 8K KV Cache) | Priority #1: 4x Continuous Batching (`--parallel 4 --cont-batching`) |
| **Cloud Heavy Brain & Sub-Agents** | `Gemini 2.5 Flash` + `DeepSeek-V3` / `Groq` (Free Tier APIs) | **Remote Cloud API** | **0 MB Local** | Parallel chapter translation lanes + Deep literary reviews |
| **CJK NLP & Glossary Engine** | `SudachiPy` (JP), `Jieba` (CN), `kiwipiepy` (KR), `FlashText` | **i7-13th Gen CPU** | **~120 MB RAM** | Pre-translation pass (~1.5s per 400-page novel) |
| **English RAG Embeddings** | `BAAI/bge-base-en-v1.5` (**ONNX INT8**, 768-dim) | **i7-13th Gen CPU** (AVX2/VNNI) | **~180 MB RAM** | Triggered per chapter **after** translation is `VERIFIED` (~80 ms/chapter) |
| **Cross-Encoder Reranker** | `BAAI/bge-reranker-v2-m3` (**ONNX INT8**) | **i7-13th Gen CPU** | **~280 MB RAM** | On-demand during active Series Chat query (~35 ms) |
| **Selective Scanned PDF OCR** | `RapidOCR` (`PP-OCRv4` ONNX) | **i7-13th Gen CPU** | **~150 MB RAM** | Triggered only on PDF pages with `< 30` selectable chars + image |
| **Pluggable TTS Audio Engine** | `Kokoro-82M` (`kokoro_onnx`) + `Edge-TTS` (`edge_tts`) + Custom HTTP (`custom_http`) | **i7-13th Gen CPU** / Cloud | **~330 MB RAM** | On-demand via Module 02's 3-sentence lookahead ring buffer |

---

## 2. Universal Coordinate System & Dual-State Sentence Schema

Original raw Asian text (Japanese/Chinese/Korean) is **never deleted or overwritten**. Every Page Card in the SQLite `pages` table stores an array of bilingual sentence objects (`sentences_json`) keyed by the **Universal Coordinate ID (`coord_id`)**:

`vol_{volume_order}_ch_{chapter_num}_pg_{page_num}_s_{sentence_num}`

### Shared `sentences_json` Item Schema (Inside SQLite `pages` Table)
```json
{
  "s_num": 3,
  "coord_id": "vol_02_ch_14_pg_08_s_03",
  "block_type": "p",
  "raw": "「カエル、影歩きのスキルを発動しろ！」",
  "translated": "\"Kael, activate your [Shadow Step] skill!\"",
  "img_path": "",
  "glossary_terms_used": ["Kael", "Shadow Step"]
}
```
* **In Module 01 (React UI):** Rendered as `<span data-coord="vol_02_ch_14_pg_08_s_03" class="reader-sentence">`.

---

## 3. Two-Tier Glossary Preview & 4-Stage Hybrid Translation Pipeline

### Stage 1: CPU CJK NLP Pre-Scan & Glossary Preview Generation (`/internal/nlp/extract-glossary`)
1. **Morphological Entity Extraction (~1.5s on CPU):** Reads raw pages from `data/novelhub.db` (read-only) and runs `SudachiPy` (`ja`), `Jieba` (`zh`), or `Kiwi` (`ko`) to extract proper nouns, character names, locations, skills, and honorifics.
2. **Global Dictionary Deduplication:** Filters out any terms already locked in SQLite's `global_glossary` or already approved (`is_approved = 1`) in `series_glossary`.
3. **Single-Shot LLM Proposal:** Sends the new unique entities (with 1 sample sentence each) to `Gemini 2.5 Flash` (fallback: local `Qwen3-8B`) to propose English translations, `entity_type` (`character`, `location`, `skill`, `item`, `organization`), and `gender_pronoun` (`he/him`, `she/her`, `they/them`, `it`, `auto`).
4. **Handoff to Glossary Preview Card:** Returns the proposed terms to Go (`Module 02`), which inserts them into `series_glossary` with `is_approved = 0` and pushes `GLOSSARY_READY` to the UI so the user can review/edit them in the **Glossary Preview Card** and click **"Confirm & Start Translation"**.

### Stage 2: Sliding-Window Context + Dynamic `FlashText` Injection (`0.2 ms` / Chunk)
Once the user confirms the glossary, Go dispatches chapters to `/internal/translate/batch`. Each chapter is segmented into **1,200-token batches (~3.5 Page Cards)**:
* **Aho-Corasick Glossary Filtering (`FlashText`):** Scans the raw 1,200-token chunk on CPU in `0.2 ms` against the combined `global_glossary` + approved `series_glossary`, injecting **only** the 8–15 terms that actually appear on those pages.
* **Source-Language Overlap Header:** Pre-attaches the **last 2 paragraphs of Batch `N-1`** in raw source language as a `[READ_ONLY_PREVIOUS_CONTEXT]` header so dialogue tone and pronouns never break across batch boundaries.

### Stage 3: Multi-Lane Execution (`provider` Assigned by Go Token-Bucket Router)
Executes the batch on the provider specified by Module 02 (`local_rtx4060`, `gemini_free`, `groq_free`, or `deepseek_cloud`):
* When running on `local_rtx4060`, calls `[http://127.0.0.1:8081/v1/chat/completions](http://127.0.0.1:8081/v1/chat/completions)` (`Qwen3-8B-Q4_K_M`) with `enable_thinking=False` (`/no_think`) for maximum translation throughput.

### Stage 4: Deterministic CPU Audit & Targeted Critic Sub-Agent (Quality Lock)
1. **Deterministic CPU Verification (`2 ms`):** Verifies every translated batch for:
   * **1:1 Sentence/Block Count Alignment:** Output `coord_id` count must match input `coord_id` count.
   * **Glossary Enforcement:** Every locked `source_term` in a sentence must contain its approved `target_term` in the translation.
   * **Zero Residual CJK Characters:** Regex scan checks for untranslated CJK characters.
2. **Targeted Critic Sub-Agent (~5–8% of Sentences):** Only sentences that fail the CPU audit are bundled into a micro-prompt and repaired by the Critic Sub-Agent. Once all sentences pass, the endpoint returns `{"status": "VERIFIED", "pages": [...]}` to Go.

---

## 4. Post-Verification Per-Series RAG Architecture

### A. Sequential Post-Verification Indexing (`/internal/rag/index-chapter`)
* **Trigger:** Called by Go (`Module 02`) immediately after a chapter's `VERIFIED` translation is committed to SQLite.
* **CPU ONNX Embedding:** Splits the verified English Page Cards into **Parent Scene Chunks (~800–900 tokens)** and **Child Micro-Chunks (~150 tokens)** tagged with `coord_id` (`vol_02_ch_14_pg_08_s_01`), and embeds them on the i7 CPU via `BAAI/bge-base-en-v1.5` (ONNX INT8) in **~80 ms per chapter**.

### B. Unified Storage Paths & Lazy `mmap` Activation
* **LanceDB Path:** `./data/lancedb/series_{series_id}.lance` (matches `Module 02` directory structure).
* **BM25 Keyword Search:** Queries the existing `pages_fts` trigram virtual table inside `./data/novelhub.db` (read-only connection) so no duplicate SQLite files are created.
* **On-Demand Hot-Swapping:** Only the currently active series' LanceDB table is memory-mapped (`mmap`); dormant series consume `0 MB` of RAM.

### C. Parent-Child Hierarchy & U-Curve Edge Positioning (`/internal/rag/query`)
1. **Spoiler-Guard Hard Filter:** Enforces `volume_order < spoiler_guard_vol OR (volume_order == spoiler_guard_vol AND page_num <= spoiler_guard_pg)`.
2. **Hybrid Search (RRF):** Combines LanceDB vector search on Child chunks + SQLite `pages_fts` BM25 search to retrieve the top 20 Child matches, then promotes them to their deduplicated **Parent Scene Chunks**.
3. **Cross-Encoder Reranker (`bge-reranker-v2-m3` ONNX INT8):** Scores candidate Parent Scenes on CPU in `~35 ms` and retains the **Top 4 Parent Scenes (~3,200 tokens)**.
4. **Edge Repositioning:** Places **Rank #1 Scene at the Top** and **Rank #2 Scene at the Bottom** of the prompt context to prevent middle-context attention dilution.

---

## 5. Multi-Format Ingestion, Export & Pluggable TTS

* **Multi-Format Ingestion (`/internal/ingest/parse`):**
  * `.epub` via `ebooklib`, `.txt` via CJK/EN chapter regex, `url` via `Trafilatura` (with Playwright fallback).
  * `.pdf` via `PyMuPDF` (`fitz`): Pages with `>= 30` selectable characters extract directly; pages with `< 30` chars + embedded image automatically run through `RapidOCR` (ONNX CPU).
* **Multi-Format Exporter (`/internal/export/build`):**
  * Generates translated or bilingual `.epub` (`ebooklib`), typeset editorial `.pdf` (`WeasyPrint`), or UTF-8 `.txt` in `./data/cache/exports/` and returns `output_file_path` for Go to stream.
* **Pluggable TTS Synthesis (`/internal/tts/stream`):**
  * Supports `engine_type` values `kokoro_onnx` (local CPU `Kokoro-82M`), `edge_tts` (free Microsoft neural cloud), and `custom_http` (forwards JSON payload to user-configured `endpoint_url` such as GPT-SoVITS or Piper).

---

## 6. Complete Bridge B Internal FastAPI Contract (`[http://127.0.0.1:8001](http://127.0.0.1:8001)`)

Matches `02_Module_2_Backend_API_Database.md` Section 8 with 100% route and payload parity:

| Internal Endpoint | Method | Input JSON Payload (From Go `:8080`) | Output / Action |
| :--- | :--- | :--- | :--- |
| `/internal/ingest/parse` | `POST` | `{"series_id", "volume_id", "volume_order", "raw_file_path", "source_format", "source_url"}` | Parses file/URL + selective `RapidOCR`, returns paginated `chapters[]` and `pages[]` with `coord_id` sentences for Go to insert. |
| `/internal/nlp/extract-glossary` | `POST` | `{"series_id", "volume_id", "source_lang"}` | Runs `SudachiPy`/`Jieba`/`Kiwi`, filters `global_glossary`, returns proposed `series_glossary` entries (`is_approved = 0`). |
| `/internal/translate/batch` | `POST` | `{"series_id", "volume_id", "chapter_num", "provider"}` | Runs `FlashText` + 2-para overlap + LLM translation + CPU NLP audit/Critic; returns `{"status": "VERIFIED", "pages": [...]}`. |
| `/internal/rag/index-chapter` | `POST` | `{"series_id", "volume_id", "chapter_num"}` | Called post-verification; indexes Parent-Child chunks into `./data/lancedb/series_{series_id}.lance`. |
| `/internal/rag/rename-entity` | `POST` | `{"series_id", "old_target_term", "new_target_term"}` | Runs word-boundary replacement across the series' LanceDB text chunks. |
| `/internal/rag/query` | `POST` | `{"series_id", "query", "spoiler_guard_vol", "spoiler_guard_pg", "rolling_summary", "recent_turns"}` | Streams SSE tokens + `citations: [{"label": "Vol 2, Page 8", "coord_id": "vol_02_ch_14_pg_08_s_03"}]`. |
| `/internal/rag/selection-action` | `POST` | `{"series_id", "coord_id", "selected_text", "action", "user_prompt", "spoiler_guard_vol", "spoiler_guard_pg"}` | Tier-0 instant bypass for inline reader `[Ask AI | Explain | Re-translate]`. |
| `/internal/rag/compact-history` | `POST` | `{"series_id", "current_summary", "turns_to_compact"}` | Compresses older chat turns into a 200-token `rolling_chat_summary`. |
| `/internal/tts/stream` | `POST` | `{"engine_id", "engine_type", "endpoint_url", "voice_id", "speed", "sentence_coord", "text"}` | Returns base64 WAV/Opus audio frame + `duration_ms` for Go's 3-sentence lookahead buffer. |
| `/internal/export/build` | `POST` | `{"volume_id", "format": "epub|pdf|txt", "mode": "translated|bilingual", "output_dir"}` | Builds exported book file and returns `{"output_file_path": "..."}`. |
