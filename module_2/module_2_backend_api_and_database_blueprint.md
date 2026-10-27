# 02_Module_2_Backend_API_Database.md
**Project Name:** NovelHub — Local-First Novel Reader, Translator & RAG Workspace  
**Module:** Module 2 — Backend API Hub, Priority Job Queue, Hybrid Router & SQLite/FTS5 Database (`:8080`)  
**Target Environment:** CachyOS Linux (Hyprland), Intel Core i7 13th Gen, 16 GB System RAM, NVIDIA RTX 4060 (8 GB VRAM)

---

## 1. Executive Summary & Architectural Role

Module 2 is the central **Hub-and-Spoke Orchestrator** of NovelHub. Written in Go (Golang) and listening on `http://127.0.0.1:8080`, it sits strictly between **Module 1 (React 19 Frontend UI on `:3000`)** and **Module 3 (Python FastAPI AI/NLP/RAG/TTS Worker on `:8001`)**.

### Core Responsibilities
1. **Strict Hub-and-Spoke Enforcement:** Module 1 never talks directly to Module 3. Every UI request, file upload, reader page fetch, inline text selection action, and WebSocket stream is managed and validated by Go.
2. **Sub-500 MB Memory Footprint:** Keeps runtime RAM usage between $60\text{ MB}$ and $140\text{ MB}$ using zero-copy file streaming (`io.TeeReader`), bounded worker pools, and SQLite memory-mapped I/O.
3. **Lock-Free Concurrent SQLite + FTS5 Storage:** Manages `data/novelhub.db` using Write-Ahead Logging (`WAL`), a serialized **Single-Connection Writer Pool**, an **8-Connection Concurrent Reader Pool**, and an FTS5 `trigram` virtual index that supports both CJK (Japanese/Chinese/Korean) characters and English substrings.
4. **3-Tier Preemptive Priority Job Queue:** Guarantees that when a user jumps to an untranslated chapter in the reader (`Tier 1`) or highlights a sentence for instant explanation/re-translation (`Tier 0`), it preempts sequential background volume translation (`Tier 2`).
5. **Post-Verification RAG Gatekeeper:** Enforces the rule that chapters are indexed into LanceDB **only after** they pass Module 3's deterministic NLP/Critic quality check (`VERIFIED` $\rightarrow$ `RAG_INDEXED`).
6. **Token-Bucket Hybrid Router:** Balances free-tier cloud APIs (**Gemini 2.5 Flash**, **Groq**, **DeepSeek-V3**, **OpenRouter**) against the local **RTX 4060 (`Qwen3-8B Q4_K_M`)** with instant automatic fallback on HTTP `429` rate limits.
7. **Single Multiplexed WebSocket Hub & 3-Sentence TTS Lookahead Buffer:** Pushes real-time chat tokens, translation progress, glossary approval cards, and pre-buffered audio frames over `ws://localhost:8080/ws/events`.

---

## 2. Go Project Directory Structure (`backend/`)

```text
novelhub/
├── backend/                          # Module 2 Root (Go 1.23+)
│   ├── cmd/
│   │   └── novelhub/
│   │       └── main.go               # Entrypoint, flags, graceful shutdown, Python supervisor
│   ├── internal/
│   │   ├── config/
│   │   │   └── config.go             # Env + SQLite user_settings loader
│   │   ├── db/
│   │   │   ├── sqlite.go             # Dual-pool (Writer=1, Reader=8) WAL + pragma setup
│   │   │   ├── migrations.go         # Embedded SQL schema + FTS5 trigram triggers
│   │   │   ├── repo_series.go        # Series, Volumes, and Spoiler-Guard CRUD
│   │   │   ├── repo_pages.go         # Paginated Page Cards & Universal Coord lookup
│   │   │   ├── repo_glossary.go      # 2-Tier Glossary & Instant SQL Character Rename
│   │   │   └── repo_settings.go      # Theme Studio & API Provider keys persistence
│   │   ├── models/
│   │   │   └── types.go              # Shared structs matching Bridge A & Bridge B JSON
│   │   ├── api/
│   │   │   ├── router.go             # go-chi/chi/v5 router, CORS, recovery middleware
│   │   │   ├── handlers_series.go    # /api/v1/series & volume ordering endpoints
│   │   │   ├── handlers_ingest.go    # Streaming multipart upload + SHA-256 deduplication
│   │   │   ├── handlers_translate.go # Pre-scan, Glossary Confirm, Priority Jump
│   │   │   ├── handlers_reader.go    # Paginated pages, Selection Actions, Spoiler-Guard
│   │   │   ├── handlers_chat.go      # Spoiler-safe RAG chat + 10-turn rolling compaction
│   │   │   ├── handlers_tts.go       # Pluggable TTS registry + 3-sentence lookahead stream
│   │   │   ├── handlers_export.go    # EPUB, PDF, TXT translated/bilingual export stream
│   │   │   └── handlers_settings.go  # Theme Studio presets & API key management
│   │   ├── queue/
│   │   │   ├── priority_heap.go      # container/heap 3-Tier Preemptive Chapter Queue
│   │   │   └── dispatcher.go         # Worker pool + Post-Verification RAG trigger
│   │   ├── hybridrouter/
│   │   │   └── token_bucket.go       # x/time/rate RPM/TPM limiters + RTX 4060 overflow
│   │   ├── ws/
│   │   │   ├── hub.go                # Multiplexed WebSocket Hub (/ws/events)
│   │   │   └── client.go             # Read/Write pumps, ping/pong heartbeats
│   │   ├── workerclient/
│   │   │   └── python_rpc.go         # HTTP client for Module 3 (127.0.0.1:8001/internal/*)
│   │   └── supervisor/
│   │       └── process.go            # Spawns & health-checks Module 3 Python worker
│   ├── go.mod
│   └── go.sum
├── data/                             # Local persistent storage (gitignored)
│   ├── novelhub.db                   # SQLite3 + FTS5 database
│   ├── novelhub.db-wal               # Write-Ahead Log file
│   ├── lancedb/                      # Memory-mapped LanceDB vector tables (Module 3)
│   ├── library/                      # Managed raw & exported novels ({series_id}/{volume_id}/)
│   └── cache/                        # Temporary OCR & TTS audio ring-buffer files
└── start.sh                          # Native CachyOS launch script
```

---

## 3. SQLite + FTS5 Database Architecture (`internal/db/`)

### 3.1 Why Dual Connection Pools + WAL Mode Are Mandatory
During a 400-page translation, up to 4 concurrent workers write translated paragraphs to SQLite while the user simultaneously scrolls through Page Cards, updates their Spoiler-Guard position, or searches via `Ctrl+K`. Standard single-pool SQLite throws `SQLITE_BUSY` errors under concurrent read/write load.

NovelHub solves this in `internal/db/sqlite.go` by opening **two distinct `sql.DB` handles** to `data/novelhub.db`:
1. **`WriterDB` (`SetMaxOpenConns(1)`):** Serializes all `INSERT`, `UPDATE`, and `DELETE` operations inside Go memory with zero lock contention.
2. **`ReaderDB` (`SetMaxOpenConns(8)`):** Serves all `SELECT` queries (`GET /pages`, `Ctrl+K` FTS5 search, glossary reads) concurrently in under $0.5\text{ ms}$.

```go
// internal/db/sqlite.go
package db

import (
	"database/sql"
	"fmt"
	_ "github.com/mattn/go-sqlite3" // Built with --tags "fts5"
)

type Store struct {
	Writer *sql.DB
	Reader *sql.DB
}

func NewStore(dbPath string) (*Store, error) {
	dsn := fmt.Sprintf("file:%s?_busy_timeout=5000&_foreign_keys=1&_journal_mode=WAL&_synchronous=NORMAL&_cache_size=-64000&_temp_store=MEMORY", dbPath)

	writer, err := sql.Open("sqlite3", dsn+"&_txlock=immediate")
	if err != nil {
		return nil, err
	}
	writer.SetMaxOpenConns(1) // Crucial: serializes writes, eliminates SQLITE_BUSY
	writer.SetMaxIdleConns(1)

	reader, err := sql.Open("sqlite3", dsn+"&mode=ro")
	if err != nil {
		return nil, err
	}
	reader.SetMaxOpenConns(8) // Concurrent non-blocking reads for UI & RAG
	reader.SetMaxIdleConns(4)

	s := &Store{Writer: writer, Reader: reader}
	if err := s.RunMigrations(); err != nil {
		return nil, err
	}
	return s, nil
}
```

---

### 3.2 Complete SQLite + FTS5 Schema (`internal/db/migrations.go`)

```sql
-- ============================================================================
-- 1. SERIES WORKSPACES (Telegram-Style Left Sidebar Threads)
-- ============================================================================
CREATE TABLE IF NOT EXISTS series (
    series_id TEXT PRIMARY KEY,
    title TEXT NOT NULL,
    source_lang TEXT NOT NULL,               -- 'ja', 'zh', 'ko', 'en'
    target_lang TEXT NOT NULL DEFAULT 'en',
    cover_path TEXT DEFAULT '',
    spoiler_guard_vol INT NOT NULL DEFAULT 1, -- Max volume_order AI Chat can access
    spoiler_guard_pg INT NOT NULL DEFAULT 1,  -- Max page_num inside spoiler_guard_vol
    rolling_chat_summary TEXT DEFAULT '',     -- Compresses chat history every 10 turns
    last_read_coord TEXT DEFAULT 'vol_01_ch_01_pg_01_s_01',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================================
-- 2. VOLUMES (Ordered Books Inside a Series Workspace)
-- ============================================================================
CREATE TABLE IF NOT EXISTS volumes (
    volume_id TEXT PRIMARY KEY,
    series_id TEXT NOT NULL REFERENCES series(series_id) ON DELETE CASCADE,
    volume_order INT NOT NULL,               -- Canonical order: 1, 2, 3...
    title TEXT NOT NULL,
    source_format TEXT NOT NULL,             -- 'epub', 'pdf', 'txt', 'url'
    source_url TEXT DEFAULT '',              -- Populated if source_format == 'url'
    raw_file_path TEXT NOT NULL,
    sha256_hash TEXT NOT NULL,               -- Prevents duplicate imports
    total_chapters INT NOT NULL DEFAULT 0,
    total_pages INT NOT NULL DEFAULT 0,
    translation_status TEXT NOT NULL DEFAULT 'RAW',
    -- Allowed: 'RAW', 'EXTRACTING_GLOSSARY', 'AWAITING_GLOSSARY_APPROVAL', 'TRANSLATING', 'COMPLETED', 'FAILED'
    translation_progress REAL NOT NULL DEFAULT 0.0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(series_id, sha256_hash)
);

CREATE INDEX IF NOT EXISTS idx_volumes_series_order ON volumes(series_id, volume_order);

-- ============================================================================
-- 3. CHAPTERS (Tracks Translation & Post-Verification RAG Indexing Per Chapter)
-- ============================================================================
CREATE TABLE IF NOT EXISTS chapters (
    chapter_id TEXT PRIMARY KEY,             -- e.g., 'vol_01_ch_14'
    volume_id TEXT NOT NULL REFERENCES volumes(volume_id) ON DELETE CASCADE,
    chapter_num INT NOT NULL,
    chapter_title_raw TEXT DEFAULT '',
    chapter_title_translated TEXT DEFAULT '',
    start_page INT NOT NULL,
    end_page INT NOT NULL,
    status TEXT NOT NULL DEFAULT 'RAW',
    -- Allowed: 'RAW', 'QUEUED', 'TRANSLATING', 'VERIFIED', 'RAG_INDEXED', 'FAILED'
    assigned_provider TEXT DEFAULT '',       -- 'local_rtx4060', 'gemini_free', 'groq_free', 'deepseek_cloud'
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(volume_id, chapter_num)
);

-- ============================================================================
-- 4. PAGINATED READER CARDS & SENTENCES (Universal Coordinate System)
-- ============================================================================
CREATE TABLE IF NOT EXISTS pages (
    page_id TEXT PRIMARY KEY,                -- e.g., 'vol_01_ch_14_pg_08'
    volume_id TEXT NOT NULL REFERENCES volumes(volume_id) ON DELETE CASCADE,
    series_id TEXT NOT NULL REFERENCES series(series_id) ON DELETE CASCADE,
    volume_order INT NOT NULL,
    chapter_num INT NOT NULL,
    page_num INT NOT NULL,                   -- 1..N sequential within volume
    is_scanned_ocr INT NOT NULL DEFAULT 0,   -- 1 if processed via RapidOCR
    sentences_json TEXT NOT NULL,
    -- JSON Array structure:
    -- [
    --   {
    --     "s_num": 1,
    --     "coord_id": "vol_01_ch_14_pg_08_s_01",
    --     "block_type": "p",        -- "p" | "h1" | "system_box" | "img"
    --     "raw": "彼は影歩を発動した。",
    --     "translated": "He activated [Shadow Step].",
    --     "img_path": ""
    --   }
    -- ]
    plain_text_search TEXT NOT NULL DEFAULT '', -- Flattened raw + translated text for FTS5
    UNIQUE(volume_id, page_num)
);

CREATE INDEX IF NOT EXISTS idx_pages_vol_pg ON pages(volume_id, page_num);
CREATE INDEX IF NOT EXISTS idx_pages_series_spoiler ON pages(series_id, volume_order, page_num);

-- ============================================================================
-- 5. FTS5 TRIGRAM VIRTUAL TABLE (Fast Ctrl+K & Hybrid BM25 Search for CJK + EN)
-- ============================================================================
CREATE VIRTUAL TABLE IF NOT EXISTS pages_fts USING fts5(
    page_id UNINDEXED,
    series_id UNINDEXED,
    volume_order UNINDEXED,
    page_num UNINDEXED,
    plain_text_search,
     content='pages',
    content_rowid='rowid',
    tokenize='trigram'
);

CREATE TRIGGER IF NOT EXISTS pages_ai AFTER INSERT ON pages BEGIN
    INSERT INTO pages_fts(rowid, page_id, series_id, volume_order, page_num, plain_text_search)
    VALUES (new.rowid, new.page_id, new.series_id, new.volume_order, new.page_num, new.plain_text_search);
END;

CREATE TRIGGER IF NOT EXISTS pages_au AFTER UPDATE OF plain_text_search ON pages BEGIN
    INSERT INTO pages_fts(pages_fts, rowid, page_id, series_id, volume_order, page_num, plain_text_search)
    VALUES ('delete', old.rowid, old.page_id, old.series_id, old.volume_order, old.page_num, old.plain_text_search);
    INSERT INTO pages_fts(rowid, page_id, series_id, volume_order, page_num, plain_text_search)
    VALUES (new.rowid, new.page_id, new.series_id, new.volume_order, new.page_num, new.plain_text_search);
END;

CREATE TRIGGER IF NOT EXISTS pages_ad AFTER DELETE ON pages BEGIN
    INSERT INTO pages_fts(pages_fts, rowid, page_id, series_id, volume_order, page_num, plain_text_search)
    VALUES ('delete', old.rowid, old.page_id, old.series_id, old.volume_order, old.page_num, old.plain_text_search);
END;

-- ============================================================================
-- 6. TWO-TIER GLOSSARY SYSTEM (Global Dictionary + Series-Locked Glossary)
-- ============================================================================
CREATE TABLE IF NOT EXISTS global_glossary (
    term_id TEXT PRIMARY KEY,
    source_lang TEXT NOT NULL,               -- 'ja', 'zh', 'ko', 'all'
    category TEXT NOT NULL,                  -- 'honorific', 'cultivation', 'skill', 'system_ui'
    source_term TEXT NOT NULL UNIQUE,
    target_term TEXT NOT NULL,
    notes TEXT DEFAULT ''
);

CREATE TABLE IF NOT EXISTS series_glossary (
    term_id TEXT PRIMARY KEY,
    series_id TEXT NOT NULL REFERENCES series(series_id) ON DELETE CASCADE,
    source_term TEXT NOT NULL,
    target_term TEXT NOT NULL,
    entity_type TEXT NOT NULL,               -- 'character', 'location', 'skill', 'item', 'organization'
    gender_pronoun TEXT NOT NULL DEFAULT 'auto', -- 'he/him', 'she/her', 'they/them', 'it', 'auto'
    sample_context TEXT DEFAULT '',          -- Example sentence shown in Glossary Preview Card
    frequency INT NOT NULL DEFAULT 1,
    is_approved INT NOT NULL DEFAULT 0,      -- 0 = Preview Pending, 1 = Locked by User
    UNIQUE(series_id, source_term)
);

CREATE INDEX IF NOT EXISTS idx_series_glossary_lookup ON series_glossary(series_id, is_approved);

-- ============================================================================
-- 7. CHAT MESSAGES (Claude/ChatGPT Thread + Clickable Coordinate Citations)
-- ============================================================================
CREATE TABLE IF NOT EXISTS chat_messages (
    msg_id TEXT PRIMARY KEY,
    series_id TEXT NOT NULL REFERENCES series(series_id) ON DELETE CASCADE,
    role TEXT NOT NULL,                      -- 'user', 'assistant', 'system_glossary_card'
    content TEXT NOT NULL,
    citations_json TEXT NOT NULL DEFAULT '[]',
    -- [{"label": "Vol 2, Ch 14, Page 8", "coord_id": "vol_02_ch_14_pg_08_s_03", "snippet": "..."}]
    spoiler_snapshot_vol INT NOT NULL DEFAULT 1,
    spoiler_snapshot_pg INT NOT NULL DEFAULT 1,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================================
-- 8. PLUGGABLE TTS ENGINES REGISTRY (NaturalReader Audio Bar)
-- ============================================================================
CREATE TABLE IF NOT EXISTS tts_engines (
    engine_id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    engine_type TEXT NOT NULL,               -- 'kokoro_onnx', 'edge_tts', 'custom_http'
    endpoint_url TEXT DEFAULT '',            -- e.g., 'http://127.0.0.1:9880/tts' for GPT-SoVITS / Piper
    default_voice TEXT NOT NULL,
    voices_json TEXT NOT NULL DEFAULT '[]',  -- Available voice profiles
    speed_default REAL NOT NULL DEFAULT 1.0,
    is_active INT NOT NULL DEFAULT 0
);

-- Seed built-in local & free cloud TTS engines
INSERT OR IGNORE INTO tts_engines (engine_id, name, engine_type, endpoint_url, default_voice, voices_json, is_active)
VALUES
    ('eng_kokoro_local', 'Kokoro-82M (Local CPU ONNX)', 'kokoro_onnx', '', 'af_heart', '["af_heart","af_bella","am_adam","bf_emma","jf_alpha"]', 1),
    ('eng_edge_cloud', 'Microsoft Edge Neural TTS (Free Cloud)', 'edge_tts', '', 'en-US-AriaNeural', '["en-US-AriaNeural","en-US-GuyNeural","en-GB-SoniaNeural"]', 0);

-- ============================================================================
-- 9. USER SETTINGS (Theme Studio Presets, Typography & Cloud API Keys)
-- ============================================================================
CREATE TABLE IF NOT EXISTS user_settings (
    setting_key TEXT PRIMARY KEY,
    setting_value_json TEXT NOT NULL,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Seed default Theme Studio & Hybrid Router settings
INSERT OR IGNORE INTO user_settings (setting_key, setting_value_json)
VALUES
    ('theme_studio', '{"active_preset":"dark","font_family":"Literata","font_size_px":18,"line_height":1.75,"page_max_width_px":760,"custom_themes":[]}'),
    ('ai_providers', '{"local_llm_url":"http://127.0.0.1:8081/v1","local_model":"Qwen3-8B-Q4_K_M","gemini_api_key":"","groq_api_key":"","deepseek_api_key":"","openrouter_api_key":"","enable_cloud_lanes":true}');
```

---

### 3.3 Instant SQL Character Rename (`PATCH /api/v1/series/{series_id}/glossary/rename`)

If a user decides after reading 3 volumes that a character named `"Kael"` should be renamed to `"Kaelen"`, re-translating 1,200 pages with an LLM would waste 20 minutes. Instead, Module 2 executes an **Instant Atomic SQL + LanceDB Rename** in under $80\text{ ms}$:

```go
// internal/db/repo_glossary.go
package db

import (
	"context"
	"fmt"
)

type RenameRequest struct {
	SeriesID      string `json:"series_id"`
	SourceTerm    string `json:"source_term"`
	OldTargetTerm string `json:"old_target_term"`
	NewTargetTerm string `json:"new_target_term"`
	NewPronoun    string `json:"new_pronoun,omitempty"`
}

func (s *Store) InstantRenameEntity(ctx context.Context, req RenameRequest) (int64, error) {
	tx, err := s.Writer.BeginTx(ctx, nil)
	if err != nil {
		return 0, err
	}
	defer tx.Rollback()

	// 1. Update locked series_glossary record
	_, err = tx.ExecContext(ctx, `
		UPDATE series_glossary
		SET target_term = ?,
		    gender_pronoun = COALESCE(NULLIF(?, ''), gender_pronoun)
		WHERE series_id = ? AND source_term = ?`,
		req.NewTargetTerm, req.NewPronoun, req.SeriesID, req.SourceTerm,
	)
	if err != nil {
		return 0, fmt.Errorf("failed updating series_glossary: %w", err)
	}

	// 2. Atomically replace across all translated Page Cards & FTS5 search text
	// Uses SQLite REPLACE on sentences_json and plain_text_search
	res, err := tx.ExecContext(ctx, `
		UPDATE pages
		SET sentences_json = REPLACE(sentences_json, ?, ?),
		    plain_text_search = REPLACE(plain_text_search, ?, ?)
		WHERE series_id = ? AND plain_text_search LIKE '%' || ? || '%'`,
		req.OldTargetTerm, req.NewTargetTerm,
		req.OldTargetTerm, req.NewTargetTerm,
		req.SeriesID, req.OldTargetTerm,
	)
	if err != nil {
		return 0, fmt.Errorf("failed updating pages JSON: %w", err)
	}

	rowsAffected, _ := res.RowsAffected()
	if err := tx.Commit(); err != nil {
		return 0, err
	}
	return rowsAffected, nil
}
```
Immediately after committing the SQLite transaction, the Go handler notifies Module 3 (`POST /internal/rag/rename-entity`) to run an in-place string replacement on the series' LanceDB text chunks.

---

## 4. 3-Tier Preemptive Priority Queue & Post-Verification RAG Gate (`internal/queue/`)

### 4.1 Priority Levels
All chapter translation tasks enter a thread-safe min-heap (`container/heap`) ordered by `(PriorityTier ASC, SequenceOrder ASC)`:
* **`Tier 0` (Immediate UI Action — Bypasses Queue):** Inline reader text selection actions (`POST /api/v1/reader/selection-action` for single-sentence re-translation or scene explanation) execute directly on a reserved fast lane without waiting behind batch jobs.
* **`Tier 1` (Reader Jump Preemption):** When a user opens an untranslated chapter in the 65% Reader Canvas, `POST /api/v1/volumes/{volume_id}/translate/prioritize` promotes that chapter (and the immediately following chapter $C+1$) to `Tier 1`. The next available worker (preferring the local RTX 4060 lane) picks it up within seconds.
* **`Tier 2` (Sequential Background Batch):** Translates all remaining chapters in canonical order (`Ch. 1, 2, 3...`).

### 4.2 Go Priority Queue Implementation

```go
// internal/queue/priority_heap.go
package queue

import (
	"container/heap"
	"sync"
	"time"
)

type PriorityTier int

const (
	TierReaderJump PriorityTier = 1 // User opened this chapter in the Reader
	TierBackground PriorityTier = 2 // Standard sequential volume translation
)

type ChapterJob struct {
	JobID       string       `json:"job_id"`       // e.g., "vol_01_ch_12"
	SeriesID    string       `json:"series_id"`
	VolumeID    string       `json:"volume_id"`
	ChapterNum  int          `json:"chapter_num"`
	Tier        PriorityTier `json:"tier"`
	EnqueuedAt  time.Time    `json:"enqueued_at"`
	HeapIndex   int          `json:"-"`
}

type jobHeap []*ChapterJob

func (h jobHeap) Len() int { return len(h) }
func (h jobHeap) Less(i, j int) bool {
	if h[i].Tier != h[j].Tier {
		return h[i].Tier < h[j].Tier // Tier 1 beats Tier 2
	}
	if h[i].VolumeID == h[j].VolumeID && h[i].ChapterNum != h[j].ChapterNum {
		return h[i].ChapterNum < h[j].ChapterNum // Canonical chapter order
	}
	return h[i].EnqueuedAt.Before(h[j].EnqueuedAt)
}
func (h jobHeap) Swap(i, j int) {
	h[i], h[j] = h[j], h[i]
	h[i].HeapIndex = i
	h[j].HeapIndex = j
}
func (h *jobHeap) Push(x any) {
	n := len(*h)
	item := x.(*ChapterJob)
	item.HeapIndex = n
	*h = append(*h, item)
}
func (h *jobHeap) Pop() any {
	old := *h
	n := len(old)
	item := old[n-1]
	old[n-1] = nil
	item.HeapIndex = -1
	*h = old[0 : n-1]
	return item
}

type PriorityQueue struct {
	mu     sync.Mutex
	cond   *sync.Cond
	items  jobHeap
	lookup map[string]*ChapterJob // Key: "vol_id_ch_num"
	closed bool
}

func NewPriorityQueue() *PriorityQueue {
	pq := &PriorityQueue{
		items:  make(jobHeap, 0),
		lookup: make(map[string]*ChapterJob),
	}
	pq.cond = sync.NewCond(&pq.mu)
	heap.Init(&pq.items)
	return pq
}

// EnqueueOrPromote inserts a new job or upgrades an existing queued chapter to Tier 1
func (pq *PriorityQueue) EnqueueOrPromote(job *ChapterJob) {
	pq.mu.Lock()
	defer pq.mu.Unlock()

	if existing, exists := pq.lookup[job.JobID]; exists {
		if job.Tier < existing.Tier && existing.HeapIndex >= 0 {
			existing.Tier = job.Tier
			existing.EnqueuedAt = time.Now()
			heap.Fix(&pq.items, existing.HeapIndex)
			pq.cond.Broadcast()
		}
		return
	}

	heap.Push(&pq.items, job)
	pq.lookup[job.JobID] = job
	pq.cond.Signal()
}

func (pq *PriorityQueue) PopBlocking() *ChapterJob {
	pq.mu.Lock()
	defer pq.mu.Unlock()

	for len(pq.items) == 0 && !pq.closed {
		pq.cond.Wait()
	}
	if pq.closed {
		return nil
	}
	job := heap.Pop(&pq.items).(*ChapterJob)
	delete(pq.lookup, job.JobID)
	return job
}
```

### 4.3 Post-Verification RAG Gate (`internal/queue/dispatcher.go`)
Per the NovelHub architecture, **RAG never indexes unverified or mid-translation drafts**. When a worker pops a `ChapterJob`:
1. Go marks the chapter `TRANSLATING` in SQLite and dispatches `POST /internal/translate/batch` to Module 3 using the provider selected by the **Token-Bucket Hybrid Router**.
2. Module 3 runs Stage 2 (FlashText Glossary + 2-paragraph overlap), Stage 3 (LLM translation), and Stage 4 (Deterministic CPU NLP tag/glossary/leftover-CJK check + targeted Critic repair).
3. Only when Module 3 returns `{"status": "VERIFIED", "pages": [...]}` does Go commit the translated sentences to SQLite, emit a `TRANSLATION_PROGRESS` WebSocket event to Chrome, and immediately trigger an asynchronous background call to `POST /internal/rag/index-chapter`.
4. Once LanceDB indexing completes, the chapter status transitions to `RAG_INDEXED`.

---

## 5. Hybrid Cloud + Local RTX 4060 Token-Bucket Router (`internal/hybridrouter/`)

To translate a 400-page novel in **6–9 minutes at \$0 cost**, Module 2 routes translation batches across free-tier cloud APIs and your local **RTX 4060 (`Qwen3-8B Q4_K_M`)**:

| Provider ID | Target Model | Default Rate Limit Bucket | Role in Hybrid Router |
| :--- | :--- | :--- | :--- |
| `local_rtx4060` | `Qwen3-8B-Q4_K_M` (`enable_thinking=False`) | Unlimited ($\sim 140\text{ tok/s}$ w/ 4x batching) | **Primary Lane for Tier 0 & Tier 1 (Reader Jump)** + automatic overflow when cloud lanes hit HTTP `429`. |
| `gemini_free` | `gemini-2.5-flash` | $15\text{ RPM}$ / $250,000\text{ TPM}$ | High-context background batch worker (`Tier 2`). |
| `groq_free` | `llama-3.3-70b-versatile` / `qwen-qwq` | $30\text{ RPM}$ / $15,000\text{ TPM}$ | Ultra-low-latency background worker & Critic repair lane. |
| `deepseek_cloud` | `deepseek-chat` (DeepSeek-V3) | Configurable RPM / TPM | High-accuracy CJK idiom & literary nuance lane. |

```go
// internal/hybridrouter/token_bucket.go
package hybridrouter

import (
	"sync"
	"time"

	"golang.org/x/time/rate"
	"novelhub/backend/internal/queue"
)

type ProviderName string

const (
	ProviderLocalRTX4060 ProviderName = "local_rtx4060"
	ProviderGeminiFree   ProviderName = "gemini_free"
	ProviderGroqFree     ProviderName = "groq_free"
	ProviderDeepSeek     ProviderName = "deepseek_cloud"
)

type ProviderLane struct {
	Name          ProviderName
	Enabled       bool
	RPMLimiter    *rate.Limiter
	CooldownUntil time.Time // Set when an HTTP 429 Too Many Requests is received
	mu            sync.Mutex
}

func (p *ProviderLane) TryAcquire() bool {
	p.mu.Lock()
	defer p.mu.Unlock()
	if !p.Enabled || time.Now().Before(p.CooldownUntil) {
		return false
	}
	return p.RPMLimiter.Allow()
}

func (p *ProviderLane) MarkRateLimited(retryAfter time.Duration) {
	p.mu.Lock()
	defer p.mu.Unlock()
	if retryAfter <= 0 {
		retryAfter = 45 * time.Second
	}
	p.CooldownUntil = time.Now().Add(retryAfter)
}

type HybridRouter struct {
	CloudLanes []*ProviderLane
	LocalLane  *ProviderLane
}

func NewHybridRouter(enableGemini, enableGroq, enableDeepSeek bool) *HybridRouter {
	return &HybridRouter{
		LocalLane: &ProviderLane{
			Name:       ProviderLocalRTX4060,
			Enabled:    true,
			RPMLimiter: rate.NewLimiter(rate.Inf, 1), // No rate limit on local GPU
		},
		CloudLanes: []*ProviderLane{
			{
				Name:       ProviderGeminiFree,
				Enabled:    enableGemini,
				RPMLimiter: rate.NewLimiter(rate.Every(4*time.Second), 2), // ~15 RPM safe bucket
			},
			{
				Name:       ProviderGroqFree,
				Enabled:    enableGroq,
				RPMLimiter: rate.NewLimiter(rate.Every(2*time.Second), 3), // ~30 RPM safe bucket
			},
			{
				Name:       ProviderDeepSeek,
				Enabled:    enableDeepSeek,
				RPMLimiter: rate.NewLimiter(rate.Every(2*time.Second), 4),
			},
		},
	}
}

// SelectProvider assigns the optimal execution lane based on job priority and token buckets
func (hr *HybridRouter) SelectProvider(tier queue.PriorityTier) ProviderName {
	// Tier 1 (Reader Jump) prefers immediate execution; check fast cloud first, else Local RTX 4060
	for _, lane := range hr.CloudLanes {
		if lane.TryAcquire() {
			return lane.Name
		}
	}
	// Seamless zero-wait overflow to Local RTX 4060 (Qwen3-8B Q4_K_M)
	return ProviderLocalRTX4060
}
```

If a cloud provider returns HTTP `429` mid-request, `workerclient` calls `lane.MarkRateLimited(45 * time.Second)` and **immediately retries that exact chunk on `ProviderLocalRTX4060`** without failing the chapter job.

---

## 6. Multiplexed WebSocket Hub & 3-Sentence TTS Lookahead (`internal/ws/`)

### 6.1 Single WebSocket Envelope (`ws://localhost:8080/ws/events`)
Every real-time push from Go to the React UI uses a unified envelope so the frontend Zustand store can route events cleanly:

```go
// internal/models/types.go
package models

type WSEventType string

const (
	EventChatStream          WSEventType = "CHAT_STREAM"
	EventTranslationProgress WSEventType = "TRANSLATION_PROGRESS"
	EventGlossaryReady       WSEventType = "GLOSSARY_READY"
	EventTTSAudioChunk       WSEventType = "TTS_AUDIO_CHUNK"
	EventChapterRAGReady     WSEventType = "CHAPTER_RAG_READY"
	EventErrorAlert          WSEventType = "ERROR_ALERT"
)

type WSEnvelope struct {
	Event     WSEventType `json:"event"`
	SeriesID  string      `json:"series_id"`
	Timestamp int64       `json:"timestamp"`
	Payload   any         `json:"payload"`
}
```

### 6.2 3-Sentence TTS Lookahead Ring Buffer (`internal/api/handlers_tts.go`)
To achieve NaturalReader-grade zero-latency playback where sentence highlighting (`<span data-coord="vol_01_ch_02_pg_04_s_03">`) never stutters between sentences:
1. When the user clicks **Play** at coordinate `vol_01_ch_02_pg_04_s_01`, Module 1 calls `POST /api/v1/tts/play` with `{"volume_id": "vol_01", "start_coord_id": "vol_01_ch_02_pg_04_s_01", "engine_id": "eng_kokoro_local", "voice_id": "af_heart", "speed": 1.0}`.
2. Go cancels any previous TTS context for that client, queries SQLite for the ordered sentence array starting at `start_coord_id`, and spawns a **Lookahead Worker (`window = 3`)**:
   * Synthesizes Sentence $S_1$ via Module 3 (`POST /internal/tts/stream`) and pushes `TTS_AUDIO_CHUNK` (`coord_id: S_1`, `audio_base64`, `duration_ms`) over the WebSocket immediately.
   * While Sentence $S_1$ plays in Chrome (typically $3\text{–}6\text{ seconds}$), the Go worker pre-generates $S_2$, $S_3$, and $S_4$ in a buffered Go channel (`make(chan TTSFrame, 3)`).
   * If the user presses `Space` (Pause) or `[ / ]` (Prev/Next Sentence), Go cancels the active `context.Context` in $< 1\text{ ms}$ so no stale audio frames are sent.

---

## 7. Complete Bridge A REST API Specification (`/api/v1/*`)

All routes are mounted in `internal/api/router.go` using `github.com/go-chi/chi/v5`.

### 7.1 Series Workspace & Ingestion Routes
* **`GET /api/v1/series`**
  * Returns all series threads for the Telegram-style left sidebar, including volume count, overall translation percentage, and `last_read_coord`.
* **`POST /api/v1/series`**
  * Body: `{"title": "Shadow Slave", "source_lang": "ja"}` $\rightarrow$ Creates workspace and `data/library/{series_id}/` folder.
* **`POST /api/v1/series/{series_id}/ingest`**
  * Supports both `multipart/form-data` (multiple `.epub`, `.pdf`, `.txt` files) and JSON (`{"url": "https://...", "source_format": "url"}`).
  * **Zero-Copy Streaming & Deduplication:** Streams each uploaded file directly to disk using `io.TeeReader(filePart, sha256Hasher)` in $64\text{ KB}$ buffers. If `sha256_hash` already exists in `volumes` for that series, returns `409 Conflict` without duplicating storage.
  * **Auto-Ordering Engine:** Extracts volume numbers from EPUB OPF `<meta property="belongs-to-collection">` / `calibre:series_index` or filename regex (`(?i)(?:vol(?:ume)?|book|part|v)[\s._-]*(\d+)`) and assigns `volume_order = 1..N`.
  * Calls Module 3 (`POST /internal/ingest/parse`) asynchronously to paginate chapters into Page Cards (`vol_{v}_ch_{c}_pg_{p}_s_{s}`) and run `RapidOCR` on any scanned PDF pages ($< 30$ selectable chars).
* **`PUT /api/v1/series/{series_id}/volumes/reorder`**
  * Body: `{"volume_ids": ["vol_02", "vol_01", "vol_03"]}` $\rightarrow$ Updates `volume_order` in a single transaction.

### 7.2 Two-Tier Glossary & Translation Control Routes
* **`POST /api/v1/volumes/{volume_id}/translate/pre-scan`**
  * Sets `translation_status = 'EXTRACTING_GLOSSARY'`.
  * Calls Module 3 (`POST /internal/nlp/extract-glossary`), which runs `SudachiPy`/`Jieba`/`Kiwi` across all pages in $\sim 1.5\text{ s}$, filters out existing `global_glossary` terms, and proposes target names + gender pronouns.
  * Inserts a `system_glossary_card` message into `chat_messages`, sets status to `'AWAITING_GLOSSARY_APPROVAL'`, and broadcasts `GLOSSARY_READY` over WebSocket.
* **`POST /api/v1/volumes/{volume_id}/translate/confirm`**
  * Body:
    ```json
    {
      "approved_series_terms": [
        {"term_id": "t_01", "source_term": "カエル", "target_term": "Kael", "entity_type": "character", "gender_pronoun": "he/him"}
      ],
      "promote_to_global": [
        {"source_lang": "ja", "category": "honorific", "source_term": "様", "target_term": "-sama"}
      ]
    }
    ```
  * Saves locked terms to `series_glossary` and `global_glossary`, transitions status to `'TRANSLATING'`, and enqueues all chapters of the volume into the `PriorityQueue` (`Tier 2`).
* **`POST /api/v1/volumes/{volume_id}/translate/prioritize`**
  * Body: `{"chapter_num": 14}` $\rightarrow$ Calls `pq.EnqueueOrPromote()` to upgrade Chapter 14 and Chapter 15 to `Tier 1 (TierReaderJump)`.
* **`PATCH /api/v1/series/{series_id}/glossary/rename`**
  * Executes `InstantRenameEntity` across SQLite `pages`, `pages_fts`, `series_glossary`, and Module 3 LanceDB.

### 7.3 Paginated Reader, Inline Selection Actions & Spoiler-Guard Routes
* **`GET /api/v1/volumes/{volume_id}/pages?page={n}&limit=5`**
  * Reads from `ReaderDB` pool; returns 5 consecutive Page Cards (`page_id`, `chapter_num`, `page_num`, and parsed `sentences_json` array with `coord_id`, `raw`, and `translated` strings).
* **`POST /api/v1/reader/selection-action`** (`Tier 0` Instant Bypass Route)
  * Body:
    ```json
    {
      "series_id": "ser_01",
      "coord_id": "vol_02_ch_14_pg_08_s_03",
      "selected_text": "彼は影歩を発動した。",
      "action": "retranslate | explain_scene | ask_ai",
      "user_prompt": "Why did he use this skill here?"
    }
    ```
  * Bypasses the batch translation queue and calls Module 3 (`/internal/rag/selection-action`) directly using `spoiler_guard_vol` and `spoiler_guard_pg` from the user's current coordinate.
* **`PUT /api/v1/series/{series_id}/spoiler-guard`**
  * Body: `{"spoiler_guard_vol": 2, "spoiler_guard_pg": 48, "last_read_coord": "vol_02_ch_14_pg_48_s_01"}` $\rightarrow$ Updates the series Spoiler-Guard ceiling in SQLite.

### 7.4 Spoiler-Safe Series Chat & Rolling 10-Turn Compaction (`POST /api/v1/series/{series_id}/chat`)
When the user sends a message in the Series Chat:
1. Go reads the series' current `spoiler_guard_vol`, `spoiler_guard_pg`, `rolling_chat_summary`, and the last 6 messages from `chat_messages`.
2. Go forwards the payload to Module 3 (`POST /internal/rag/query`) and streams the SSE tokens back to Chrome as `CHAT_STREAM` WebSocket events, ending with the clickable `citations` array (`[{"label": "Vol 2, Page 14", "coord_id": "vol_02_ch_03_pg_14_s_02"}]`).
3. **Automatic 10-Turn Memory Compaction:** Every 10 messages in `chat_messages`, Go triggers a lightweight background call to Module 3 (`POST /internal/rag/compact-history`) that compresses older chat turns into a 200-token `rolling_chat_summary` in the `series` table, preventing chat history from diluting RAG attention.

### 7.5 Multi-Format Export & Settings Routes
* **`POST /api/v1/volumes/{volume_id}/export`**
  * Body: `{"format": "epub | pdf | txt", "mode": "translated | bilingual"}`
  * Calls Module 3 (`POST /internal/export/build`) which generates the file on disk (`ebooklib` for EPUB, `WeasyPrint` for typeset PDF, or clean UTF-8 `.txt`), then Go streams the file back to Chrome with `http.ServeFile` and `Content-Disposition: attachment`.
* **`GET /api/v1/settings` & `PUT /api/v1/settings`**
  * Reads and persists **Theme Studio** presets (`Dark`, `Sepia`, `Paper`, `E-Ink`, custom fonts/spacing) and **AI Provider Keys** in `user_settings`, hot-reloading the `HybridRouter` lanes without restarting the server.

---

## 8. Bridge B Internal RPC Client (`internal/workerclient/python_rpc.go`)

Go communicates with Module 3 (`http://127.0.0.1:8001`) over a persistent HTTP/1.1 keep-alive connection pool configured for zero-copy local file paths:

| Internal Module 3 Endpoint | Triggered By (Go Module 2) | Timeout | Payload Summary |
| :--- | :--- | :--- | :--- |
| `POST /internal/ingest/parse` | `handlers_ingest.go` | $5\text{ min}$ | `{"series_id", "volume_id", "raw_file_path", "source_format", "source_url"}` |
| `POST /internal/nlp/extract-glossary` | `handlers_translate.go` | $60\text{ s}$ | `{"series_id", "volume_id", "source_lang"}` |
| `POST /internal/translate/batch` | `queue/dispatcher.go` | $3\text{ min}$ | `{"series_id", "volume_id", "chapter_num", "provider"}` |
| `POST /internal/rag/index-chapter` | `queue/dispatcher.go` (Post-Verify) | $60\text{ s}$ | `{"series_id", "volume_id", "chapter_num"}` |
| `POST /internal/rag/rename-entity` | `handlers_translate.go` | $15\text{ s}$ | `{"series_id", "old_target_term", "new_target_term"}` |
| `POST /internal/rag/query` | `handlers_chat.go` (SSE Stream) | $2\text{ min}$ | `{"series_id", "query", "spoiler_guard_vol", "spoiler_guard_pg", "rolling_summary", "recent_turns"}` |
| `POST /internal/rag/selection-action` | `handlers_reader.go` (`Tier 0`) | $45\text{ s}$ | `{"series_id", "coord_id", "selected_text", "action", "user_prompt", "spoiler_guard_vol", "spoiler_guard_pg"}` |
| `POST /internal/tts/stream` | `handlers_tts.go` | $30\text{ s}$ | `{"engine_id", "voice_id", "speed", "sentence_coord", "text"}` |
| `POST /internal/export/build` | `handlers_export.go` | $3\text{ min}$ | `{"volume_id", "format", "mode", "output_dir"}` |

---

## 9. Native CachyOS Process Supervisor & Startup (`cmd/novelhub/main.go` & `start.sh`)

To keep local execution effortless on CachyOS Hyprland without Docker overhead, the Go binary includes a built-in child-process supervisor (`internal/supervisor/process.go`) that automatically spawns and watches the Python `:8001` worker and cleanly terminates the entire process group on `Ctrl+C` (`SIGINT`/`SIGTERM`).

### `start.sh` (Root Launch Script for CachyOS)
```bash
#!/usr/bin/env bash
set -euo pipefail

echo "===================================================="
echo "  Starting NovelHub Local Workspace (CachyOS)       "
echo "  Hardware Profile: i7 13th Gen | 16GB RAM | RTX4060"
echo "===================================================="

mkdir -p data/library data/lancedb data/cache

# 1. Start local llama-server for Qwen3-8B (Q4_K_M) with 4x Continuous Batching on RTX 4060
if ! pgrep -x "llama-server" > /dev/null; then
  if [ -f "./models/Qwen3-8B-Q4_K_M.gguf" ]; then
    echo "[1/3] Launching llama-server (Qwen3-8B Q4_K_M, 4x Continuous Batching) on :8081..."
    llama-server \
      -m ./models/Qwen3-8B-Q4_K_M.gguf \
      --host 127.0.0.1 --port 8081 \
      -ngl 99 \
      -c 8192 \
      --parallel 4 \
      --cont-batching \
      --cache-type-k q8_0 --cache-type-v q8_0 \
      > data/cache/llama_server.log 2>&1 &
    LLAMA_PID=$!
  else
    echo "[1/3] Note: ./models/Qwen3-8B-Q4_K_M.gguf not found yet; running with Cloud/Configured lanes."
  fi
fi

# 2. Launch Go Backend Hub (:8080), which supervises Module 3 Python Worker (:8001)
echo "[2/3] Starting Module 2 Go Hub (:8080) & Module 3 Python AI Worker (:8001)..."
cd backend
go run ./cmd/novelhub/main.go \
  --port 8080 \
  --db ../data/novelhub.db \
  --library ../data/library \
  --python-worker-dir ../ai_worker \
  --spawn-python=true
```