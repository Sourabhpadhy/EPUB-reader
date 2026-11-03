# NovelHub

**Read a whole series, translate it consistently, and chat with an AI that only knows what you have read.**

NovelHub is a local-first reader and translator for multi-volume novels (light novels, web novels, ebooks). It runs on your own machine, keeps your library private, and uses cloud models only if you turn them on.

> **Status: pre-alpha.** The design is complete and implementation is starting (Phase 0). The commands below describe the intended experience and will be finalised as each phase ships. See the [roadmap](#roadmap).

<!-- TODO: add screenshots here (use public-domain text only) -->
<!-- TODO: add badges (CI, license, PyPI) once the repo is public -->

---

## Why NovelHub

| Problem with other tools | What NovelHub does |
|---|---|
| Translating volume by volume makes names and honorifics drift | A per-series glossary locks names, genders and terms across every volume |
| Asking an AI about the plot spoils the story | The AI only sees text up to **your** reading position |
| Cloud-only tools see everything you read | Local by default; cloud lanes are opt-in per series |
| Fixing a character name means re-translating | A rename takes milliseconds |

---

## Features

**Library and reader**
* Series workspaces with reorderable volumes
* Import `.epub` and `.txt` (PDF with selective OCR planned)
* Paginated reader: vertical scroll or horizontal flip, light-novel callout boxes and images kept
* Raw, translated or bilingual view; click any translated sentence to peek at the source
* Four themes with light customization (accent colour, font, size, line height, width, paper tone)
* `Ctrl+K` search across series, volumes, scenes and notes, including 2-character CJK names

**Translation**
* Two-tier glossary (global dictionary plus per-series) with an approval card before translation starts
* Local model by default; add any number of cloud lanes with your own keys
* Quality checks plus targeted repair before a chapter is accepted
* Per-series style guide (honorifics, tone, sound effects)
* Corrections memory: your manual edits become examples for later chapters
* Opening an untranslated chapter moves it to the front of the queue

**Notes and AI chat**
* Highlights, notes and bookmarks anchored to exact sentences
* Spoiler-safe chat with citations that jump straight to the passage
* **Notes → chat:** turn selected notes into a conversation, and save any answer back as a note
* "Who is this?" character cards and a "Previously on…" recap, both spoiler-safe

**Audio, export and data**
* Text-to-speech with live sentence highlighting (Kokoro local; Edge-TTS optional)
* Export translated or bilingual `.epub`, `.pdf` and `.txt`
* One-file backup and restore per series

The full list, with behaviour details and ideas that are not committed yet, is in [`docs/FEATURES_AND_IDEAS.md`](docs/FEATURES_AND_IDEAS.md).

---

## How the spoiler guard works

Every sentence has a permanent ID. NovelHub stores one position per series, the last sentence the AI may know, and applies it to every source of context: search results, chapter summaries, character facts, your notes, recaps, and chat history. If you move the guard backwards, later material disappears from the AI's view immediately.

---

## Requirements

| Item | Requirement |
|---|---|
| OS | Linux (primary). Windows and macOS are best effort |
| Python | 3.12 |
| Local model | Any OpenAI-compatible endpoint: `llama-server`, Ollama, LM Studio or vLLM. Or use cloud lanes only |
| GPU | Optional. It only affects how fast the local model runs |

The reference development machine is an Intel i7 (13th gen), 16 GB RAM and an RTX 4060 (8 GB VRAM). Measured speed and memory numbers will be published here after the Phase 0 benchmarks, with the hardware named.

---

## Install (planned)

```bash
uv tool install "novelhub[all]"      # or: pipx install "novelhub[all]"
novelhub doctor                      # checks your setup and tells you what to fix
novelhub start                       # opens http://127.0.0.1:8080
```

The base install is small. Heavy features are optional extras:

| Extra | Adds |
|---|---|
| `ja`, `zh`, `ko` | Name detection for Japanese, Chinese, Korean |
| `ocr` | OCR for scanned PDFs |
| `rag` | Embeddings and reranking for AI chat |
| `kokoro` | Local text-to-speech |
| `edge` | Edge-TTS (unofficial endpoint; may break) |
| `all` | Everything above |

A Docker image will be offered as a secondary option.

---

## Quick start

1. Run `novelhub start`. The first-run wizard finds your local model endpoint (or lets you enter one) and runs a short test translation.
2. Create a series and drop in your `.epub` or `.txt` files. Check the volume order.
3. Review the glossary card (character names, genders, key terms) and click **Confirm & Start Translation**.
4. Start reading. Translation continues in the background and jumps ahead to wherever you open the book.
5. Select text to ask the AI, save a note, or fix a translation. Open the chat to ask questions about what you have read so far.

---

## Privacy

* Binds to `127.0.0.1` and requires a token
* No telemetry, no accounts
* No network calls except to endpoints you configure
* Book text goes to a cloud model only if you enable cloud use for that series
* API keys are stored in your OS keyring and are never included in backups or returned by the API

---

## Legal notice

NovelHub does not host, share or distribute books or translations, and it ships no scraping tools. You are responsible for having the right to use the files you import, and translations are intended for personal use. Check each cloud provider's data terms before enabling a cloud lane.

---

## Roadmap

| Phase | Focus | Status |
|---|---|---|
| 0 | Foundations, benchmarks, repo setup | Planned |
| 1 | Library, reader, notes, themes, search, setup wizard | Planned |
| 2 | Glossary, local translation, style guide, corrections | Planned |
| 3 | Multiple cloud lanes | Planned |
| 4 | Spoiler-safe chat, notes → chat, character cards, recap | Planned |
| 5 | PDF + OCR, export, backup and restore | Planned |
| 6 | Text-to-speech | Planned |
| 7 | Packaging and v1.0 release | Planned |

Not in the core, but designed for: web-novel URL importers and custom TTS engines (as plugins), and an advanced theme editor.

---

## Extending NovelHub

Sources, TTS engines and exporters are plugin interfaces discovered through Python entry points (`novelhub.sources`, `novelhub.tts`, `novelhub.export`). A plugin guide will live in `docs/`.

---

## Development

<!-- TODO: finalise these commands in Phase 0 -->

```bash
git clone https://github.com/Sourabhpadhy/novelhub.git   # repo to be renamed from EPUB-reader
cd novelhub/backend && uv sync
cd ../frontend && npm install && npm run build
cd ../backend && uv run novelhub start --dev
```

| Folder | Contents |
|---|---|
| `contracts/` | OpenAPI file and event schemas (the API source of truth) |
| `backend/` | Python app (FastAPI) |
| `frontend/` | React app |
| `eval/` | Public-domain test fixtures and a mock LLM server |
| `docs/` | Architecture and feature docs |

The tests run without a GPU by using the mock LLM server.

---

## Documentation

* [`docs/00_Master_Overview_and_Connections.md`](docs/00_Master_Overview_and_Connections.md): architecture, data model, phases
* [`docs/FEATURES_AND_IDEAS.md`](docs/FEATURES_AND_IDEAS.md): feature behaviour, deferred items, ideas

---

## Contributing

Contributions are welcome, especially plugins, test fixtures, and platform testing on Windows and macOS. See `CONTRIBUTING.md` (coming in Phase 0). Look for the `good first issue`, `help wanted` and `plugin` labels.

---

## License

<!-- TODO: confirm before the first public commit. Recommended: Apache-2.0 -->
To be decided before the first public release.

---

## Built with

llama.cpp, FastAPI, SQLite, LanceDB, ONNX Runtime, SudachiPy, Jieba, kiwipiepy, Kokoro, React.
