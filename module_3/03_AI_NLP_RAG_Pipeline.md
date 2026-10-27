# Module 03: AI/ML, NLP, Hybrid SLM & Per-Series RAG Pipeline

**File:** `03_AI_NLP_RAG_Pipeline.md`  
**Target System:** CachyOS Linux (Hyprland) | Intel Core i7-13th Gen | 16 GB System RAM | NVIDIA RTX 4060 (8 GB VRAM)  
**Architecture Mode:** Native Localhost Web App (`http://localhost:8080`, Cloud-Deployable via `.env`)  
**Scope:** CJK NLP Glossary Extraction, Hybrid Multi-Agent Translation, Post-Translation Per-Series RAG, Smart Task Routing, and Pluggable TTS Audio.

---

## 1. Execution Priority & Hardware Resource Split

### Core Architectural Rule: Translation-First Pipeline
When a novel or volume is imported, **100% of compute focuses on translation first**. RAG vector indexing never runs on raw untranslated text during import; instead, each chapter is indexed on the CPU **only after** its English translation is verified. This prevents CPU/GPU resource contention, eliminates double-indexing disk writes, and guarantees maximum translation speed.

| Subsystem | Model / Library | Assigned Hardware | Memory Budget | Execution Timing |
| :--- | :--- | :--- | :--- | :--- |
| **Local Translation & Chat SLM** | `Qwen3-8B` (`Q4_K_M`, `/no_think` for translation, `/think` for offline review) or `Gemma-3-9B-It` via `llama-server` | **RTX 4060 GPU (100% Dedicated)** | **~6.3 GB VRAM** (Weights + 8K KV Cache) | Priority #1: 4x Continuous Batching (`--parallel 4 --cont-batching`) |
| **Cloud Heavy Brain & Sub-Agents** | `Gemini 2.5 Flash` + `DeepSeek-V3` / `Groq` (Free Tier APIs) | **Remote Cloud API** | **0 MB Local** | Parallel chapter translation lanes + Deep literary reviews |
| **CJK NLP & Glossary Engine** | `SudachiPy` (JP), `Jieba` (CN), `kiwipiepy` (KR), `FlashText` | **i7-13th Gen CPU** | **~120 MB RAM** | Pre-translation pass (~1.5s per 400-page novel) |
| **English RAG Embeddings** | `BAAI/bge-base-en-v1.5` (**ONNX INT8**, 768-dim) | **i7-13th Gen CPU** (AVX2/VNNI) | **~180 MB RAM** | Triggered per chapter **after** translation finishes (~80 ms/chapter) |
| **Cross-Encoder Reranker** | `BAAI/bge-reranker-v2-m3` (**ONNX INT8**) | **i7-13th Gen CPU** | **~280 MB RAM** | On-demand during active Series Chat query (~35 ms) |
| **Selective Scanned PDF OCR** | `RapidOCR` (`PP-OCRv4` ONNX) | **i7-13th Gen CPU** | **~150 MB RAM** | Triggered only on PDF pages with `< 30` chars + embedded image |
| **Pluggable TTS Audio Engine** | `Kokoro-82M` (ONNX) + `Edge-TTS` + Custom HTTP Webhook | **i7-13th Gen CPU** / Cloud | **~330 MB RAM** | On-demand 3-sentence lookahead ring buffer |

---

## 2. Dual-State Text Storage (Permanent Raw Preservation + English Translation)

The original raw Asian text (Japanese/Chinese/Korean) is **never deleted or overwritten**. Every paragraph or DOM block is stored as a paired bilingual record in SQLite (`book_nodes`), allowing the user to click any translated sentence in the UI to inspect the original raw text or re-translate specific paragraphs.

### Shared Paragraph Node Schema (`book_nodes`)
```json
{
  "node_id": "series_04_vol_02_ch_14_p_08",
  "series_id": "series_04",
  "book_id": "vol_02",
  "book_order": 2,
  "chapter_num": 14,
  "page_num": 52,
  "dom_tag": "p",
  "epub_cfi": "epubcfi(/6/28!/4/2/16/1:0)",
  "raw_text": "「カエル、影歩きのスキルを発動しろ！」",
  "translated_text": "\"Kael, activate your Shadow Step skill!\"",
  "translation_status": "VERIFIED",
  "glossary_terms_used": ["Kael", "Shadow Step"]
}
```

---

## 3. One-Go High-Quality Translation Pipeline (Zero Review Popups)

Translation executes automatically in one continuous run upon book import without interrupting the user with confirmation popups. Extracted character names, genders, and lore terms are saved directly into the Series Chat memory.

### Stage 1: CPU CJK NLP Pre-Scan & Auto-Glossary Lock (~1.5 Seconds)
1. **Morphological Entity Extraction:** Python worker detects the source language (`ja`, `zh`, `ko`) and runs `SudachiPy`, `Jieba`, or `Kiwi` across the raw novel to extract proper nouns, character names, honorifics (`-san`, `-sama`, `Shixiong`, `Hyung`), and recurring skill/realm terms.
2. **One-Shot Glossary Resolution:** The deduplicated entity list (paired with 1 sample context sentence per entity) is sent in a **single request** to the Cloud Heavy Brain (`Gemini 2.5 Flash`, fallback: local `Qwen3-8B`).
3. **Persistent Series Chat Glossary (`series_glossary` Table):**
   ```json
   {
     "series_id": "series_04",
     "source_term": "カエル",
     "target_term": "Kael",
     "category": "CHARACTER",
     "pronouns": "he/him",
     "speech_style": "stoic, concise",
     "first_seen_book": 1
   }
   ```

### Stage 2: Sliding-Window Context + Dynamic `FlashText` Injection (0.2 ms/Chunk)
Each chapter is segmented into **1,200-token batches (~3.5 pages)** along clean `<p>` boundaries:
* **Aho-Corasick Glossary Filtering (`FlashText`):** Scans the raw 1,200-token chunk on CPU in `0.2 ms` and injects **only** the 8–15 glossary entries that actually appear in that batch. This cuts prompt token overhead by 85% and prevents attention dilution.
* **Source-Language Overlap Header:** Pre-attaches the **last 2 paragraphs of Chunk `N-1`** in raw source language as a `[READ_ONLY_PREVIOUS_CONTEXT]` block. This preserves mid-scene dialogue flow and pronouns while allowing all chunks to be translated in parallel.

### Stage 3: Heterogeneous Multi-Lane Parallel Sub-Agents
The Go Orchestrator dispatches prepared chunks concurrently across four execution lanes:
* **Priority Lane 0 (Active Reader Preemption):** If the user opens an untranslated chapter in the Reader UI, the **Local RTX 4060** immediately preempts background tasks and streams that chapter to the screen in ~25 seconds.
* **Worker Lane 1 (Local RTX 4060 GPU):** `Qwen3-8B` (`Q4_K_M`, `enable_thinking=False`) running 4x continuous batching (`~140+ tok/s` combined local throughput).
* **Worker Lanes 2 & 3 (Free Cloud Sub-Agents):** `Gemini 2.5 Flash` and `Groq / OpenRouter` Free Tiers process background chapters concurrently up to their free RPM/TPM token-bucket limits. Any HTTP 429 rate-limit response automatically overflows the chunk back to Worker Lane 1 with zero failures.

### Stage 4: Deterministic CPU Audit & Targeted Critic Sub-Agent (Quality Lock)
1. **Deterministic CPU Verification (2 ms):** Checks every translated chunk for:
   * **1:1 DOM Tag Alignment:** Input `<p>` count must equal output `<p>` count.
   * **Glossary Enforcement:** Every locked `source_term` present in the raw paragraph must have its exact `target_term` in the translated paragraph.
   * **Zero Residual CJK Characters:** Scans via regex for untranslated character blocks.
2. **Targeted Critic Sub-Agent (~5–8% of Paragraphs):** Paragraphs that pass the CPU audit are marked `VERIFIED` immediately. Only paragraphs that fail a glossary check, tag check, or length-ratio check are routed to the **Critic Sub-Agent** (`Gemini 2.5 Flash` or local `Gemma-3-9B-It`) for surgical repair.

---

## 4. Post-Translation Per-Series On-Demand RAG Architecture

### A. Sequential Post-Translation Indexing
* **Trigger:** As soon as a chapter reaches `translation_status = "VERIFIED"`, a low-priority CPU background thread splits the verified English text into **Parent-Child chunks** and embeds them via `bge-base-en-v1.5` (ONNX INT8) in **~80 ms per chapter** (~2.5 seconds total per 400-page novel).

### B. Physical Folder Isolation & Lazy `mmap` Activation
* **Directory Structure:** `~/.local/share/novel-workspace/rag/series_{series_id}/`
  * `lancedb_vectors/` (Partitioned by `book_order`)
  * `fts5_keywords.sqlite` (BM25 exact keyword and quote index)
* **On-Demand Hot-Swapping:**
  * **Dormant Series:** Use **0 MB of RAM** (stored purely on SSD).
  * **Active Series:** Opening a Series Chat memory-maps (`mmap`) **only** that series' LanceDB and FTS5 files (~15–40 MB RAM). Switching to another Series Chat unmaps the previous series immediately.

### C. Parent-Child Hierarchy & U-Curve Edge Positioning (Anti-Dilution)
To eliminate "Lost in the Middle" attention dilution across 10–30+ volumes:
1. **Spoiler-Guard Hard Filter:** Before searching, restricts partitions to `book_order < user_current_book OR (book_order = user_current_book AND page_num <= user_current_page)`.
2. **Child Micro-Chunk Hybrid Search (~150 tokens / 1–2 paragraphs):** Queries `LanceDB` (semantic vectors) + `SQLite FTS5` (BM25 exact names/skills) and combines results via Reciprocal Rank Fusion (RRF) to fetch the top 20 child matches.
3. **Parent Scene Expansion (~800 tokens / ~3 pages):** Maps the top 20 child chunks to their deduplicated Parent Scene blocks so the LLM receives full scene context.
4. **Cross-Encoder Reranker (`bge-reranker-v2-m3` ONNX INT8):** Scores the candidate Parent Scenes against the user's query on the i7 CPU in ~35 ms and keeps only the **Top 4 Parent Scenes (~3,200 tokens)**.
5. **Edge Repositioning in Prompt (~6,000–7,500 tokens total):** Places **Rank #1 Scene at the Top** and **Rank #2 Scene at the Bottom** of the retrieved context block to align with the LLM's peak primacy and recency attention zones.

---

## 5. Smart Task Router: Local SLM + Free Cloud Brain

To overcome the literary analysis limitations of an 8B local model while keeping bulk processing free and fast:

| Task Type | Primary Assigned Model | Offline / Rate-Limit Fallback | Prompt Context Structure |
| :--- | :--- | :--- | :--- |
| **Bulk Chapter Translation** | Hybrid Parallel (`RTX 4060 Qwen3-8B /no_think` + Free Cloud Lanes) | `RTX 4060 Qwen3-8B` (`enable_thinking=False`) | 1,200-token batch + `FlashText` glossary + 2-para raw overlap |
| **Chapter Summary** | `RTX 4060 Qwen3-8B` (`enable_thinking=False`) | `RTX 4060 Qwen3-8B` (`enable_thinking=False`) | Verified English chapter (~3,500 tokens) + Character Glossary |
| **Deep Book Review / Character Critique / Multi-Book Plot Q&A** | **Cloud Heavy Brain** (`DeepSeek-V3` / `Gemini 2.5 Flash`) | `RTX 4060 Qwen3-8B` (**`enable_thinking=True`**) | Pinned Character/Lore Memory + Chapter Synopsis Map + Top 4 Reranked Parent Scenes |
| **In-Chat Bulk Name Edit** (*"Rename Kael to Kaelen"*) | Deterministic SQLite/FTS5 Update + Chat Confirmation | Local Deterministic Update | Updates `series_glossary` and runs instant CPU regex replacement across `translated_text` |

---

## 6. Pluggable NaturalReader TTS Audio Pipeline

* **Sentence-Span Indexing:** Every translated sentence inside a Reader Page Card is wrapped in `<span data-sent-id="ch14-p8-s2">`.
* **3-Sentence Lookahead Ring Buffer:** When the user clicks **Play**, the Python worker synthesizes Sentence `N` immediately while pre-generating Sentences `N+1` and `N+2` in RAM for zero-latency transitions and synchronized sentence highlighting.
* **Pluggable TTS Provider Contract:**
  * `kokoro_local`: Runs `Kokoro-82M` ONNX on the i7 CPU (~330 MB RAM, 54 voices).
  * `edge_cloud`: Streams neural read-aloud audio via `edge-tts` WebSocket (zero API key required).
  * `custom_webhook`: Sends `{ "text": "...", "voice_id": "...", "speed": 1.0 }` to any user-configured local endpoint (e.g., `GPT-SoVITS`, `Fish Speech`, or `Piper TTS`) registered via the UI's **"Add Audio Reader"** modal.

---

## 7. Internal Python ML Worker API Contract (`http://127.0.0.1:8001`)

Exposes internal endpoints consumed by the Go Orchestrator (Module 02):

| Endpoint | Method | Payload / Action | Response |
| :--- | :--- | :--- | :--- |
| `/ml/ingest/parse` | `POST` | `{ "file_path": "...", "series_id": "...", "book_id": "..." }` — Runs `ebooklib` / `PyMuPDF` + selective `RapidOCR` (<30 chars/page). | `{ "status": "parsed", "total_pages": 412, "ocr_pages_processed": 18 }` |
| `/ml/nlp/extract-glossary` | `POST` | `{ "series_id": "...", "book_id": "..." }` — Runs CJK NER + 1-shot LLM glossary lock into `series_glossary`. | `{ "glossary_count": 142, "status": "locked" }` |
| `/ml/translate/batch` | `POST` | `{ "chunk_id": "...", "worker_lane": "local_gpu | gemini_free | groq_free" }` — Runs `FlashText` injection, translation, CPU audit, and Critic pass if needed. | `{ "chunk_id": "...", "status": "VERIFIED", "critic_triggered": false }` |
| `/ml/rag/index-chapter` | `POST` | `{ "series_id": "...", "book_id": "...", "chapter_num": 1 }` — Called **after** chapter translation completes; embeds Parent-Child chunks on CPU. | `{ "indexed_scenes": 14, "elapsed_ms": 78 }` |
| `/ml/rag/query` | `POST` | `{ "series_id": "...", "query": "...", "spoiler_book": 2, "spoiler_page": 52, "task_type": "qa | review" }` — Hot-swaps `mmap` index, runs Hybrid Search + Reranker + LLM stream. | Server-Sent Events (SSE) token stream + `citations: [{ "book_order": 2, "page_num": 19, "node_id": "..." }]` |
| `/ml/tts/synthesize` | `POST` | `{ "engine_id": "kokoro_local", "sentence_id": "ch14-p8-s2", "text": "...", "voice": "af_heart", "speed": 1.0 }` | Audio WAV/MP3 binary stream + duration metadata |
