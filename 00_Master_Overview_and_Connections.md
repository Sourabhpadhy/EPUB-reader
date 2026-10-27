# 00_Master_Overview_and_Connections.md
**Project Name:** NovelHub — Local-First Series ePub/PDF/Web Novel Reader, Translator & RAG Workspace

---

## 1. Executive Summary & Core Vision
SeriesHub AI is a local-first web application (hosted locally and accessed via Chrome, architected to be 100% cloud-deployable later) designed for reading, translating, listening to, and querying multi-volume book series (primarily Asian Light Novels and Web Novels).

* **Telegram-Style Navigation:** Left sidebar manages "Series Workspaces" (each chat thread represents one book series containing Volumes 1..N in canonical order).
* **Claude/ChatGPT-Style Series Onboarding & Chat:** Clicking the bottom `+` icon opens an AI Chatbot start screen where users drop `.epub`, `.pdf`, `.txt` files or paste Web Novel URLs, preview/approve the extracted glossary, and ask spoiler-safe questions across the entire series.
* **Sophisticated PDF-Style Paginated Reader:** Renders chapters as clean, discrete editorial page cards (`Page X of Y`) with click-to-peek source text and inline illustrations.
* **NaturalReader-Style Pluggable Audio Bar:** Bottom docked audio player with real-time sentence highlighting and pluggable TTS engines (`Kokoro-82M` local ONNX, `Edge-TTS`, or custom local endpoints).

---

## 2. Target Hardware & Resource Allocation Contract
* **OS & Environment:** CachyOS Linux (Hyprland Wayland), Native Localhost execution (`./start.sh`), Chrome Browser UI.
* **CPU & System RAM Budget (Intel Core i7 13th Gen, 16 GB RAM):**
  * **Go Orchestrator + SQLite + LanceDB (`mmap`):** Under 500 MB RAM.
  * **Python AI/NLP/TTS Worker (ONNX CPU Runtime):** Under 2.0 GB RAM. Runs CJK NLP (`SudachiPy`, `Jieba`, `Kiwi`), `RapidOCR` (for 20–40p scanned PDF sections), and `Kokoro-82M` TTS strictly on CPU threads so GPU VRAM is never starved.
* **GPU Budget (NVIDIA GeForce RTX 4060, 8 GB VRAM):**
  * Dedicated 100% to Local SLM (`Qwen3-8B` or `Sakura-7B-Qwen2.5` at `Q4_K_M`, `enable_thinking=False`, ~6.37 GB VRAM with 8K context and 4x continuous batching) + `bge-reranker-v2-m3`.

---

## 3. System Topology & Port Assignments (Hub-and-Spoke)
**Strict Rule:** Module 1 (UI) NEVER communicates directly with Module 3 (AI Worker). All traffic routes through Module 2 (Go Backend Hub).

1. **Module 1 — Frontend UI/UX (`http://localhost:3000` in Dev / Served by Go in Prod):**
   * Tech: React 19, TypeScript, Tailwind CSS, Zustand, TanStack Virtual.
2. **Module 2 — Backend API, Job Queue & Database Hub (`[http://127.0.0.1:8080](http://127.0.0.1:8080)`):**
   * Tech: Go (Golang), Chi/Fiber Router, Gorilla WebSocket, SQLite3 (with FTS5), Hybrid Cloud/Local Token-Bucket Queue.
3. **Module 3 — AI, NLP, RAG, Ingestion & TTS Micro-Worker (`[http://127.0.0.1:8001](http://127.0.0.1:8001)`):**
   * Tech: Python 3.12 (`uv`), FastAPI, `PyMuPDF` + `RapidOCR`, `ebooklib`, `Trafilatura` (Web URL scraper), `WeasyPrint` (PDF exporter), `LanceDB`, `SudachiPy`/`Jieba`/`Kiwi`, `FlashText`, `pykokoro` / `edge-tts`.

---

## 4. The Universal Coordinate System (Shared Across All 3 Modules)
Every paragraph and sentence in every imported volume is assigned a deterministic **Coordinate ID (`coord_id`)**:

`vol_{volume_order}_ch_{chapter_num}_pg_{page_card_num}_s_{sentence_num}`
*(Example: `vol_02_ch_14_pg_08_s_03` = Volume 2, Chapter 14, Page Card 8, Sentence 3)*

* **In Module 1 (UI):** Every sentence in the Paginated Reader renders as `<span data-coord="vol_02_ch_14_pg_08_s_03">`. Clicking a RAG citation badge scrolls straight to `pg_08`. Pressing Play on the Audio Bar highlights `s_01`, `s_02`, `s_03` in sequence.
* **In Module 2 (DB):** Stored as the primary lookup key for pages, translations, and Spoiler-Guard progress (`user_current_coord`).
* **In Module 3 (RAG & TTS):** Stored in every LanceDB vector chunk and returned in every TTS audio timestamp frame.

---

## 5. Multi-Format Import & Export Contract
* **Supported Imports (`POST /api/v1/series/{series_id}/ingest`):**
  1. `.epub`: Unpacked via `ebooklib`; preserves inline images and CSS callout boxes.
  2. `.pdf`: Inspected page-by-page via `PyMuPDF`. Pages with greater than 30 selectable characters extract instantly; scanned pages (20–40p sections) automatically route to `RapidOCR` (CPU ONNX).
  3. `.txt`: Auto-detects chapter headings via CJK/English regex (`Chapter X`, `第X章`, `제X화`) and paginates into ~300-word Page Cards.
  4. `Web URL`: Uses `Trafilatura` (with `Playwright` fallback for JS sites) to fetch either a single chapter or a Table-of-Contents index link list and builds a structured volume.
* **Supported Exports (`POST /api/v1/volumes/{volume_id}/export`):**
  * Formats: `epub` (reflowable translated or bilingual ePub), `pdf` (sophisticated typeset PDF via `WeasyPrint`), or `txt` (clean formatted UTF-8 text).

---

## 6. Two-Tier Glossary & Preview Workflow Contract
To guarantee translation accuracy and speed across 400–1,000 pages:
1. **Tier 1 — Global Dictionary (`global_glossary` table):** Applies across *all* series chats automatically. Stores common honorifics (`-san`, `-sama`, `Shixiong`, `Hyung`), recurring genre terms (cultivation realms, LitRPG system brackets `[Skill: ...]`), and preferred translation rules.
2. **Tier 2 — Series Glossary (`series_glossary` table):** Scoped to one `series_id`. Stores character names, locked gender pronouns (`he/him`, `she/her`), locations, and unique skills.
3. **Mandatory Glossary Preview State Machine:**
   * Step 1: User requests translation of Volume(s) -> Status becomes `EXTRACTING_GLOSSARY`.
   * Step 2: Module 3 runs CPU NLP (`SudachiPy`/`Jieba`/`Kiwi`) across all 400 pages in ~1.5s, filters out terms already in `global_glossary`, and asks the LLM to propose translations + gender pronouns for new entities -> Status becomes `AWAITING_GLOSSARY_APPROVAL`.
   * Step 3: Module 1 (UI) displays an interactive **Glossary Preview Card** inside the Series Chat. User can edit/approve terms (or promote general terms to the Global Dictionary) and click **"Confirm & Start Translation"**.
   * Step 4: Status becomes `TRANSLATING` -> Hybrid Router (Free Cloud APIs + Local RTX 4060 `Qwen3-8B` with 2-paragraph sliding window overlap) translates chapters in priority order.

---

## 7. Shared SQLite + FTS5 + LanceDB Database Schema (Owned by Module 2 & 3)

### A. SQLite Tables (`data/serieshub.db`)
```sql
-- 1. Series Workspaces (Telegram Sidebar Threads)
CREATE TABLE series (
    series_id TEXT PRIMARY KEY,
    title TEXT NOT NULL,
    source_lang TEXT NOT NULL,          -- 'ja', 'zh', 'ko', 'en', etc.
    target_lang TEXT DEFAULT 'en',
    cover_path TEXT,
    spoiler_guard_vol INT DEFAULT 1,    -- Max volume AI chat is allowed to access
    spoiler_guard_pg INT DEFAULT 1,     -- Max page card inside spoiler_guard_vol
    rolling_chat_summary TEXT DEFAULT '', -- Compresses chat every 10 turns
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 2. Ordered Volumes Inside a Series
CREATE TABLE volumes (
    volume_id TEXT PRIMARY KEY,
    series_id TEXT NOT NULL REFERENCES series(series_id) ON DELETE CASCADE,
    volume_order INT NOT NULL,          -- 1, 2, 3... (auto-detected or user reordered)
    title TEXT NOT NULL,
    source_format TEXT NOT NULL,        -- 'epub', 'pdf', 'txt', 'url'
    raw_file_path TEXT NOT NULL,
    total_pages INT DEFAULT 0,
    translation_status TEXT DEFAULT 'RAW', -- 'RAW', 'EXTRACTING_GLOSSARY', 'AWAITING_GLOSSARY_APPROVAL', 'TRANSLATING', 'COMPLETED'
    translation_progress REAL DEFAULT 0.0
);

-- 3. Paginated Reader Cards & Sentences
CREATE TABLE pages (
    page_id TEXT PRIMARY KEY,           -- e.g., 'vol_01_ch_02_pg_05'
    volume_id TEXT NOT NULL REFERENCES volumes(volume_id) ON DELETE CASCADE,
    chapter_num INT NOT NULL,
    chapter_title TEXT,
    page_num INT NOT NULL,              -- 1..N within volume
    sentences_json TEXT NOT NULL        -- JSON array: [{"s_num": 1, "coord_id": "vol_01_ch_02_pg_05_s_01", "raw": "...", "translated": "...", "type": "p|callout|img"}]
);

-- 4. Two-Tier Glossaries (Global + Series)
CREATE TABLE global_glossary (
    term_id TEXT PRIMARY KEY,
    source_lang TEXT NOT NULL,
    category TEXT NOT NULL,             -- 'honorific', 'cultivation', 'skill', 'system_ui'
    source_term TEXT NOT NULL UNIQUE,
    target_term TEXT NOT NULL,
    notes TEXT
);

CREATE TABLE series_glossary (
    term_id TEXT PRIMARY KEY,
    series_id TEXT NOT NULL REFERENCES series(series_id) ON DELETE CASCADE,
    source_term TEXT NOT NULL,
    target_term TEXT NOT NULL,
    entity_type TEXT NOT NULL,          -- 'character', 'location', 'skill', 'item'
    gender_pronoun TEXT DEFAULT 'auto', -- 'he/him', 'she/her', 'they/them', 'it', 'auto'
    is_approved INT DEFAULT 0,          -- 0 = Preview Pending, 1 = Locked by User
    UNIQUE(series_id, source_term)
);

-- 5. Chat Messages (Claude/ChatGPT Thread)
CREATE TABLE chat_messages (
    msg_id TEXT PRIMARY KEY,
    series_id TEXT NOT NULL REFERENCES series(series_id) ON DELETE CASCADE,
    role TEXT NOT NULL,                 -- 'user', 'assistant', 'system_glossary_card'
    content TEXT NOT NULL,
    citations_json TEXT DEFAULT '[]',   -- [{"label": "Vol 2, Page 45", "coord_id": "vol_02_ch_04_pg_45_s_01"}]
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 6. Pluggable TTS Engines Registry
CREATE TABLE tts_engines (
    engine_id TEXT PRIMARY KEY,
    name TEXT NOT NULL,                 -- 'Kokoro-82M (Local ONNX)', 'Edge-TTS (Free Cloud)', 'Custom Endpoint'
    engine_type TEXT NOT NULL,          -- 'kokoro_onnx', 'edge_tts', 'custom_http'
    endpoint_url TEXT,                  -- Used when engine_type == 'custom_http'
    default_voice TEXT NOT NULL,
    is_active INT DEFAULT 0
);
```

### B. LanceDB Vector Schema (`data/lancedb/series_chunks.lance`)
* `chunk_id` (`string`): Unique parent/child ID
* `series_id` (`string`): Filter key
* `volume_order` (`int32`): Used for Spoiler-Guard filtering (`volume_order <= spoiler_guard_vol`)
* `page_num` (`int32`): Used for Spoiler-Guard filtering (`page_num <= spoiler_guard_pg`)
* `coord_id` (`string`): Exact jump coordinate (`vol_02_ch_14_pg_08_s_01`)
* `chunk_level` (`string`): `'child'` (150 tokens for search) or `'parent'` (900 tokens for LLM context) or `'chapter_summary'`
* `parent_id` (`string`): Links child chunk to parent scene chunk
* `entities` (`list<string>`): Character/skill names present in chunk (extracted via CPU NLP)
* `text` (`string`): Indexed text
* `vector` (`float32[768]`): Dense semantic embedding

---

## 8. Inter-Module API & WebSocket Contracts

### Bridge A: Module 1 (React UI) <-> Module 2 (Go Backend `:8080`)

#### REST Endpoints
* `GET /api/v1/series` -> Lists all series threads for the Telegram left sidebar.
* `POST /api/v1/series` -> Creates a new series workspace.
* `POST /api/v1/series/{series_id}/ingest` -> Accepts multipart files (`.epub`, `.pdf`, `.txt`) or JSON `{"url": "https://..."}`; returns sorted `volumes[]`.
* `PUT /api/v1/series/{series_id}/volumes/reorder` -> Updates canonical `volume_order`.
* `POST /api/v1/volumes/{volume_id}/translate/pre-scan` -> Triggers CPU NLP glossary extraction and returns the `Glossary Preview` list.
* `POST /api/v1/volumes/{volume_id}/translate/confirm` -> Saves user-approved `series_glossary` & promoted `global_glossary` items, then starts the Hybrid Translation Queue.
* `POST /api/v1/volumes/{volume_id}/translate/prioritize` -> Bumps a specific `chapter_num` to Priority #1 when a user jumps ahead in the reader.
* `GET /api/v1/volumes/{volume_id}/pages?page={n}&limit=5` -> Fetches paginated reader cards and sentence coordinate arrays.
* `PUT /api/v1/series/{series_id}/spoiler-guard` -> Updates `{"spoiler_guard_vol": 2, "spoiler_guard_pg": 45}`.
* `POST /api/v1/series/{series_id}/chat` -> Sends user query; streams response via WebSocket `/ws/events`.
* `POST /api/v1/volumes/{volume_id}/export` -> Body: `{"format": "epub|pdf|txt", "mode": "translated|bilingual"}` -> Returns downloadable file stream.
* `GET /api/v1/tts/engines` & `POST /api/v1/tts/engines` -> Lists or registers a new pluggable audio reader engine.

#### Multiplexed WebSocket (`ws://localhost:8080/ws/events`)
All real-time updates between Module 2 and Module 1 use this single JSON envelope:
```json
{
  "event": "CHAT_STREAM | TRANSLATION_PROGRESS | GLOSSARY_READY | TTS_AUDIO_CHUNK",
  "series_id": "ser_01",
  "payload": {
    "volume_id": "vol_01",
    "coord_id": "vol_01_ch_01_pg_02_s_04",
    "delta_text": "...",
    "progress_pct": 42.5,
    "audio_base64": "...",
    "citations": [{"label": "Vol 1, Page 12", "coord_id": "vol_01_ch_01_pg_12_s_02"}]
  }
}
```

### Bridge B: Module 2 (Go `:8080`) <-> Module 3 (Python AI Worker `:8001`)
* `POST /internal/ingest/parse` -> Input: `{"volume_id", "file_path_or_url", "source_format"}` -> Parses `.epub`/`.pdf`/`.txt`/URL, runs `RapidOCR` on scanned pages, populates `pages` table, indexes LanceDB Parent-Child chunks.
* `POST /internal/nlp/extract-glossary` -> Input: `{"series_id", "volume_id", "source_lang"}` -> Runs `SudachiPy`/`Jieba`/`Kiwi` + single-shot LLM proposal -> Populates unapproved `series_glossary` rows.
* `POST /internal/translate/batch` -> Input: `{"series_id", "volume_id", "chapter_num", "provider": "local_rtx4060 | gemini_free | groq_free"}` -> Runs `FlashText` dynamic glossary filtering + 2-paragraph sliding window translation + CPU NLP tag/glossary verification.
* `POST /internal/rag/query` -> Input: `{"series_id", "query", "spoiler_guard_vol", "spoiler_guard_pg"}` -> Runs Metadata Filter -> Hybrid BM25 + LanceDB Child Search -> Parent Scene Promotion -> `bge-reranker` -> Edge Repositioning -> Streams answer + `coord_id` citations.
* `POST /internal/tts/stream` -> Input: `{"engine_id", "voice_id", "speed", "sentences": [{"coord_id", "text"}]}` -> Streams WAV/Opus chunks tagged with `coord_id`.
* `POST /internal/export/build` -> Input: `{"volume_id", "format": "epub|pdf|txt", "mode": "translated|bilingual"}` -> Builds file on disk via `ebooklib` or `WeasyPrint` and returns `output_file_path`.

---

## 9. Scope Boundaries for Your 3 Module Chats

* **Chat 1 (`01_Module_1_Frontend_UI_UX.md`):** Build ONLY the React + TypeScript + Tailwind UI: Telegram Series Sidebar, AI Chatbot Start/Dropzone View, Interactive Glossary Preview Card, Sophisticated PDF-Style Paginated Reader Canvas, Click-to-Peek Source Popover, Export Modal, and NaturalReader Docked Audio Bar with Pluggable TTS Settings. Must consume Bridge A REST + WebSocket contracts.
* **Chat 2 (`02_Module_2_Backend_API_Database.md`):** Build ONLY the Go Server & SQLite/FTS5 Manager: File watcher (`3–4 GB` library), SQLite schema migrations, WebSocket Hub, Priority Chapter Job Queue, and the Token-Bucket Hybrid Router (balancing Free Cloud API rate limits with Local RTX 4060 workers). Must implement Bridge A & Bridge B contracts.
* **Chat 3 (`03_Module_3_AI_NLP_RAG_TTS.md`):** Build ONLY the Python FastAPI AI Worker: Multi-format Ingestion (`PyMuPDF` + `RapidOCR`, `ebooklib`, `Trafilatura`), Multi-format Exporter (`WeasyPrint`, `ebooklib`), CJK NLP Glossary Extractor (`SudachiPy`/`Jieba`/`Kiwi` + `FlashText`), `Qwen3-8B` / Free-Cloud Translation Prompt Pipeline + Critic Verifier, Parent-Child LanceDB RAG with Edge Repositioning, and Pluggable TTS Adapter (`Kokoro-82M` ONNX + `Edge-TTS` + Custom HTTP).
