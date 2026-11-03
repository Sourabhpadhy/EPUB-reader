# FEATURES_AND_IDEAS.md

**Project:** NovelHub · **Version:** 1.0 · **Date:** 2026-11-03
**Companion to:** `00_Master_Overview_and_Connections.md` (architecture and phases). This file is the product view: what NovelHub does, how each feature should behave, what is deferred, and what is only an idea.

**Status legend:** `Planned` = in the phase plan · `Deferred` = designed for, not built · `Idea` = not committed

---

## 1. Feature catalogue

### Library and ingest

| ID | Feature | Behaviour | Phase | Status |
|---|---|---|---|---|
| F1 | Series workspaces | Sidebar of series; each has ordered volumes you can reorder; cover, title, languages | 1 | Planned |
| F2 | File ingest | `.epub`, `.txt` first, `.pdf` with selective OCR later; SHA-256 dedup; volume order suggested from titles | 1, 5 | Planned |
| F2b | Web-novel URL import | Not in core. A `SourceAdapter` plugin can add it later; the interface and SSRF-safe fetcher exist from Phase 0 | — | Deferred |

### Reader

| ID | Feature | Behaviour | Phase | Status |
|---|---|---|---|---|
| F3 | Paginated reader | Vertical scroll or horizontal flip; page cards; callout boxes and images kept; reader shows raw, translated or bilingual | 1 | Planned |
| F4 | Sentence-level coordinates | Every sentence has a stable ID; progress, notes, citations, TTS and search all reference it | 1 | Planned |
| F16 | Global search (`Ctrl+K`) | Searches series, volumes, scenes and notes; works for 2-character CJK names | 1 | Planned |
| F17 | Themes | 4 presets plus light customization: accent colour, font family, font size, line height, content width, paper tone | 1 | Planned |

### Notes and study

| ID | Feature | Behaviour | Phase | Status |
|---|---|---|---|---|
| F18 | Highlights, notes, bookmarks | Select text to highlight (colours), add a note, tag it, or bookmark a spot; a notes panel filters by volume, tag, kind and search | 1 | Planned |
| F19 | Notes → chat | Turn selected notes into a chat thread; save assistant answers as notes (details in §2.1) | 4 | Planned |
| F15b | Notes export | Export notes as Markdown or JSON | 5 | Planned |

### Glossary and translation

| ID | Feature | Behaviour | Phase | Status |
|---|---|---|---|---|
| F5 | Source peek | Click a translated sentence to see the raw text and the glossary terms used | 2 | Planned |
| F6 | Two-tier glossary | Global dictionary plus per-series glossary; entity types, gender pronouns; approval card for the core terms before translation; other terms tentative | 2 | Planned |
| F7 | Hybrid translation | Local lane plus any number of cloud lanes you configure; one lane per chapter; per-series switch for cloud use | 2, 3 | Planned |
| F8 | Quality lock | Coverage and residual-CJK checks; targeted Critic repair; units that still fail are flagged for review | 2 | Planned |
| F9 | Reader-jump priority | Opening an untranslated chapter promotes it and the next one | 2 | Planned |
| F10 | Selection actions | Ask AI, Explain scene, Re-translate, Add to glossary, Save as note | 2, 4 | Planned |
| F11 | Instant rename | Rename a character everywhere in milliseconds; gender changes offer a targeted re-translate | 2 | Planned |
| F22 | Per-series style guide | Honorifics, tone, sound effects, custom rules injected into every prompt (details in §2.4) | 2 | Planned |
| F23 | Corrections memory | Your manual edits become examples for later batches (details in §2.5) | 2, 3 | Planned |

### AI chat and spoiler safety

| ID | Feature | Behaviour | Phase | Status |
|---|---|---|---|---|
| F12 | Spoiler-safe chat | Multiple threads per series; the AI sees only text up to your read position (or a position you set); answers cite sentence ranges that jump into the reader | 4 | Planned |
| F13 | Memory compaction | Each thread keeps a rolling summary of older turns | 4 | Planned |
| F20 | "Who is this?" card | Tap a character for facts you already know, first appearance and relationships (§2.2) | 4 | Planned |
| F21 | "Previously on…" recap | Short recap up to your position when you return to a series (§2.3) | 4 | Planned |

### Audio, export and data

| ID | Feature | Behaviour | Phase | Status |
|---|---|---|---|---|
| F14 | Text-to-speech | Kokoro (local) and Edge-TTS (optional extra, unofficial endpoint); sentence highlighting while playing | 6 | Planned |
| F14b | Custom TTS engines | HTTP or other engines through the `TtsEngine` plugin interface | — | Deferred |
| F15 | Export | Translated or bilingual `.epub`, `.pdf`, `.txt` | 5 | Planned |
| F25 | Backup and restore | One archive per series (data and files; vectors optional because they can be rebuilt); keys never included (§2.7) | 5 | Planned |

### Setup and distribution

| ID | Feature | Behaviour | Phase | Status |
|---|---|---|---|---|
| F24 | First-run wizard and `novelhub doctor` | Finds or connects a local LLM endpoint, checks optional extras, shows model licenses, tests a sample translation (§2.6) | 0, 1 | Planned |
| F26 | Install and distribution | PyPI package with bundled UI, optional extras, CLI, Docker image, install guides, docs | 7 | Planned |

---

## 2. Behaviour details for the newer features

### 2.1 Notes → chat (F19)

**Story:** "I highlighted twenty things about the villain across three volumes. I want to talk about them."

1. Open the notes panel, select notes (or filter by volume, tag or kind), press **Chat about these notes**.
2. A new thread is created, seeded with the notes and the passages they point at (plus a little surrounding text), all limited to what your spoiler guard allows.
3. The seed notes stay pinned during the whole thread. If they do not all fit, the most recent or relevant are kept and the UI tells you how many were left out.
4. Suggested first prompts: "Summarize my notes", "What questions are still open?", "Find connections between these notes", or type your own.
5. **Save as note** on any assistant message creates a note anchored to the answer's first citation (or your current position if there is none).

**Done when:** a thread created from 10 notes answers using them; notes beyond the guard are never included; a saved answer shows up in notes search.

### 2.2 "Who is this?" card (F20)

Tap or hover a glossary name in the reader. The card shows target and source name, entity type, pronoun, a link to the first appearance, and a list of facts from chapters you have read, each with a citation. Facts come from chapter indexing, so opening a card is instant. Nothing from beyond your position appears.

**Done when:** a character introduced in a later volume shows no card facts until you reach that point.

### 2.3 "Previously on…" recap (F21)

When you reopen a series after a break (threshold is a setting) or press **Recap**, you get a short summary built from the last few chapter summaries before your position plus your latest notes. It is cached until your position changes.

**Done when:** the recap never mentions anything after your read position, and moving the guard backward regenerates it.

### 2.4 Per-series style guide (F22)

A small form per series: honorifics (keep, translate, localize), tone (literal, natural, casual), sound effects (translate, keep, romanize), and a short free-text box. It is added to every translation prompt. Changing it never rewrites finished chapters by surprise; the UI offers **Re-translate affected chapters**.

### 2.5 Corrections memory (F23)

Edit any translated sentence. NovelHub stores your version as a new revision and as a correction (source, machine text, your text). In later batches it picks up to three relevant corrections and shows them to the model as examples. Your manual edits are never overwritten by an automatic pass.

**Done when:** after fixing how a recurring phrase is rendered, the next chapter uses your wording.

### 2.6 First-run wizard and `novelhub doctor` (F24)

Steps: choose languages → detect a local endpoint (llama-server, Ollama, LM Studio) or enter one → optionally add cloud lanes (keys go to the OS keyring) → check installed extras (OCR, NLP language packs, TTS) → run a short sample translation to confirm everything works → show model licenses and the content disclaimer. `novelhub doctor` runs the same checks from the terminal and prints what to fix.

### 2.7 Backup and restore (F25)

A backup is a single archive with a manifest (schema version, series ID), the series' database rows, its source files, and optionally vectors. API keys are never included. Restore validates the manifest, extracts to a staging area, and then imports; IDs are immutable, so a restore on a fresh machine reproduces the same references.

---

## 3. Deferred, but designed for

| Item | How the door stays open |
|---|---|
| Web-novel URL import | `SourceAdapter` plugin interface (`novelhub.sources`), shared SSRF-safe fetcher, snapshot files for reproducible parsing. Core ships no network source because of legal and maintenance cost |
| Custom TTS engines | `TtsEngine` plugin interface (`novelhub.tts`) |
| Advanced Theme editor (full CSS-variable editing, import/export of themes) | Themes are stored as variables in settings, so a richer editor only adds UI |
| More export formats | `Exporter` interface (`novelhub.export`) |
| Native LLM provider adapters | The generic OpenAI-compatible lane handles most; adapters can be added behind the same lane config |

---

## 4. Idea backlog (not committed)

| Idea | Why it might be worth it |
|---|---|
| Export highlighted words or sentences to Anki | Useful for language learners; uses notes you already have |
| Tap-to-lookup dictionary (for example JMdict for Japanese) | Common in CJK readers; check dictionary licenses first |
| Shareable glossary packs (JSON) | Community-built name lists per series; names only, no book text |
| Reading stats and goals | Light motivation: pages per day, time per series |
| Translation diff view (machine vs your edit) | Makes the corrections memory visible |
| Phone or tablet reading over the local network | Needs authentication design before it is safe |
| Plugin gallery in the README | Helps contributors share adapters and engines |
| UI language selection | Broader adoption for an open-source project |
| Prompt and glossary "recipes" per genre | Better first-time quality for LitRPG, xianxia, romance |

---

## 5. Non-goals

* Hosting, sharing or distributing books or translations
* Bundling books or model weights in the repository
* DRM removal
* Multi-user cloud hosting (today the app is single-user and local)
* Telemetry or accounts

---

## 6. Open decisions

| Decision | Options | Needed by |
|---|---|---|
| License | Apache-2.0 (recommended) or AGPL-3.0 | Before the first public commit |
| Package and repo name | Confirm "NovelHub" is free on GitHub and PyPI, or pick another | Before the first release |
| Platform support level | Linux only at v1.0, or Linux + Windows + macOS | Phase 7 |
| Default recommended local model | Decided by the Phase 0 benchmark | Phase 0 |
| Embedding and reranker models | Decided by the Phase 0 benchmark | Phase 0 |
