# 01_Module_1_Frontend_UI_UX.md
**Module:** Module 1 — Frontend UI/UX & Client State Architecture  
**Project:** SeriesHub AI (Local-First Series ePub/PDF/Web Novel Reader, Translator & RAG Workspace)  
**Target Environment:** Chrome Browser (Linux / CachyOS Hyprland Desktop), `< 200 MB` RAM footprint  
**Upstream Contract:** Strictly adheres to `00_Master_Overview_and_Connections.md` (Bridge A: REST `/api/v1` + WebSocket `/ws/events` on `:8080`)

---

## 1. Frontend Tech Stack & Libraries
* **Core Framework:** React 19 + TypeScript + Vite (Fast HMR, zero-bundle overhead in dev, static build served by Go in prod).
* **Styling & Primitives:** Tailwind CSS v4 + `shadcn/ui` (Radix UI primitives for accessible modals, popovers, dropdowns) + `lucide-react` icons.
* **Layout & Split Panes:** `react-resizable-panels` (Smooth draggable dividers between Sidebar, Chat, and Reader with persistent width memory).
* **Animations:** `motion` (`framer-motion`) for Telegram sidebar collapse, split-screen slide-ins, floating selection pills, and horizontal page-flip transitions.
* **Virtualization (Memory Guard):** `@tanstack/react-virtual` (Renders only the 2–3 visible Page Cards in the DOM at any time so 1,000-page novels never lag Chrome).
* **State Management:** `Zustand` (3 modular stores: `useWorkspaceStore`, `useReaderStore`, `useAudioStore`) + `TanStack Query` (for REST caching).

---

## 2. The 5 Core Views & Layout States

### State A: Left Navigation Sidebar (Telegram Desktop Style)
* **Expanded Mode (280px default when Reader is closed):**
  * **Top Bar:** Global Search input (`Ctrl+K` to search across all series/volumes) + Theme Studio button.
  * **Series Thread List:** Scrollable list of Series Workspaces. Each item shows: Series Cover thumbnail, Series Title, Volume count badge (`Vol. 1–8`), active translation progress ring (if translating), and last active timestamp.
  * **Bottom Floating Action:** Prominent `+ New Series Chat` button anchored at the bottom corner.
* **Auto-Collapsed Mode (64px icon strip when Reader is open):**
  * Automatically collapses to circular Series Cover avatars when the Right-Pane Book Reader opens, maximizing horizontal space for Side-by-Side Chat + Book reading. Expands temporarily on hover or via `Ctrl+B`.

### State B: "Start a Series Chat" Onboarding View (Claude / ChatGPT Style)
* **Trigger:** Clicking `+ New Series Chat` at the bottom of the Telegram sidebar.
* **Center Canvas:**
  * **Multi-Format Dropzone Card:** Accepts drag-and-drop of multiple `.epub`, `.pdf`, `.txt` files at once, plus an inline URL input field for pasting Web Novel Table-of-Contents or chapter links.
  * **Interactive Series Order Bar:** As soon as files are dropped, displays horizontal draggable pills (`[1. Vol 01] ➔ [2. Vol 02] ➔ [3. Vol 03]`) auto-sorted by metadata/filename. Users can drag to reorder or click to rename before confirming.
  * **Language & Glossary Selector:** Dropdown for Source Language (`Japanese`, `Chinese`, `Korean`, `Auto-Detect`) and toggle for `Apply Global Dictionary (Honorifics & Genre Terms)`.
  * **Bottom Chat Prompt Bar:** Allows immediate starter instructions (e.g., *"Extract glossary and translate Volume 1"*).

### State C: Active Series Chat View (Book Closed — Main Center View)
* **Top Pinned Header — The Series Bookshelf:**
  * Horizontal scrollable shelf locked at the top of the chat displaying every volume in canonical order (`Book 1`, `Book 2`, `Book 3`...).
  * Each Book Card displays: Cover art, Title, Status Badge (`RAW`, `GLOSSARY PENDING`, `TRANSLATING 65%`, `READY`), an **Export** icon (`.epub`/`.pdf`/`.txt`), and a primary **`Read Book`** button.
* **Center Scrollable Thread — AI Chat & Interactive Cards:**
  * **Interactive Glossary Preview Card (`role: system_glossary_card`):** Rendered inline before a translation starts. Displays an editable table of extracted character names, locked gender pronouns (`he/him`, `she/her`, `they/them`), and skill terms. Includes a checkbox per row to **"Save to Global Dictionary"** and a primary **`Confirm & Start Translation`** button.
  * **Spoiler-Safe Q&A Messages:** User questions and streamed AI answers. Every scene reference renders as a clickable citation pill (`[Vol 2 • Ch 14 • Page 8]`). Clicking the pill automatically transitions the workspace into **State D (Split-Screen)** and scrolls straight to that exact page and sentence.
* **Bottom Input Dock:**
  * Chat textarea + **Spoiler-Guard Indicator Pill** (`Memory Locked: Up to Vol 2, Page 45`) so the user always sees what the AI is allowed to know.

### State D: Split-Screen Reader + Synced Side Chat (Book Open)
* **Layout:** Resizable 2-Pane Split (`35% Left: Synced Series Chat` | `65% Right: Paginated Book Reader`), with the Telegram Sidebar collapsed into a 64px rail on the far left.
* **Synchronized Chat Behavior:**
  * The Left Pane contains the exact same Series Chat (Bookshelf compacted at the top + full message history below).
  * Questions asked while the book is open—or triggered by highlighting text inside the book—appear immediately in this chat thread and remain in the main chat history when the book is closed.
* **Right Pane — Sophisticated PDF-Style Paginated Reader Canvas:**
  * **Top Reader Toolbar:**
    * Volume & Chapter dropdown jumper.
    * **Inline Live Translation Pill:** (`Translating Ch. 8/24 • 65% — Current Chapter Ready`). If the user jumps to an untranslated chapter, automatically fires `POST /api/v1/volumes/{id}/translate/prioritize`.
    * **View Mode Toggle:** Switch between **Vertical Scroll** (continuous PDF-style stacked page cards with crisp gaps) and **Horizontal Flip** (1-page or 2-page book spread).
    * **Spoiler-Guard Sync Button:** 1-click lock icon (`Sync AI Memory to Current Page`).
    * **Maximize Reader Toggle:** Collapses the Side Chat into a floating mini-button when full-screen focus is desired.
  * **Editorial Page Cards (`max-width: 760px`):**
    * Distinct page containers with subtle elevation, top running headers (`Series Title • Chapter Name`), and bottom footers (`Page X of Y`).
    * Preserves Light Novel callout boxes (`[System Alert / Skill Window]`) and inline illustrations.
    * Every sentence is wrapped in `<span data-coord="vol_02_ch_14_pg_08_s_03" class="reader-sentence">`.
  * **Interactive Text Features:**
    * **Click-to-Peek Source Popover:** Clicking any translated sentence opens a floating popover showing the raw CJK/source text + active glossary terms used in that sentence.
    * **Selection Action Pill:** Highlighting any text pops up a quick action bar: `[Ask AI in Chat]`, `[Explain Scene]`, `[Re-translate]`, `[+ Add to Glossary]`.

### State E: NaturalReader Docked Audio Bar & Pluggable TTS Modal
* **Bottom Floating Dock (Inside Reader Pane):**
  * Play / Pause (`Space`), Previous/Next Sentence (`[`, `]`), Playback Speed slider (`0.8x` to `2.5x`), Active Voice selector (`Kokoro-82M Local`, `Edge-TTS Neural`, or custom engines), and an **`+ Add Audio Engine`** button.
* **Real-Time Sentence Highlighting:**
  * As `TTS_AUDIO_CHUNK` events arrive over WebSocket with a `coord_id`, the matching `<span data-coord="...">` receives the `.tts-active-sentence` highlight styling, and the reader automatically smooth-scrolls or flips to the next Page Card when reaching the bottom of a page.
* **Pluggable TTS Modal:**
  * Allows registering new local or remote TTS engines by specifying `Engine Name`, `Type (Kokoro ONNX / Edge-TTS / Custom HTTP Endpoint)`, `Endpoint URL (e.g., http://localhost:9880/tts)`, and `Default Voice ID`.

---

## 3. Dynamic Theme Studio & Custom CSS Variable Contract
The Theme Studio Drawer supports 4 built-in reading presets plus full custom user theming via CSS root variables (`localStorage` + `theme.json` import/export):

| CSS Variable | OLED Dark (Default) | Warm Sepia | Paper Light | E-Ink Grayscale | Custom User Picker |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `--bg-workspace` | `#0B0D13` | `#EAE0C8` | `#F1F5F9` | `#D4D4D4` | User Hex Picker |
| `--bg-page-card` | `#131722` | `#F4ECD8` | `#FFFFFF` | `#F5F5F5` | User Hex Picker |
| `--text-reader` | `#E2E8F0` | `#3B2F2F` | `#1E293B` | `#111111` | User Hex Picker |
| `--accent-primary` | `#6366F1` | `#B45309` | `#2563EB` | `#000000` | User Hex Picker |
| `--tts-highlight` | `rgba(99,102,241,0.25)` | `rgba(180,83,9,0.22)` | `rgba(37,99,235,0.18)` | `rgba(0,0,0,0.15)` | User RGBA Picker |
| `--reader-font` | `'Inter' / 'Lexend'` | `'Literata' / Serif` | `'Merriweather'` | `'Atkinson Hyperlegible'` | Font Family + Size (`14px`–`26px`) + Line Height (`1.4`–`2.2`) |

---

## 4. Component Hierarchy (Directory Structure)
```text
frontend/
├── src/
│   ├── components/
│   │   ├── sidebar/
│   │   │   ├── TelegramSidebar.tsx         # Collapsible left series list + Ctrl+K search
│   │   │   └── SeriesThreadItem.tsx        # Cover avatar, title, vol count, progress badge
│   │   ├── onboarding/
│   │   │   ├── NewSeriesStartView.tsx      # Chatbot-style welcome + multi-file/URL dropzone
│   │   │   └── VolumeOrderPillBar.tsx      # Drag-and-drop canonical book order bar
│   │   ├── chat/
│   │   │   ├── SeriesChatContainer.tsx     # Shared between Center View & Split-Screen Left Pane
│   │   │   ├── SeriesBookshelfHeader.tsx   # Top horizontal book shelf (Book 1..N in order)
│   │   │   ├── GlossaryPreviewCard.tsx     # Interactive glossary table + Global Dict promotion
│   │   │   ├── ChatMessageList.tsx         # Streamed responses + clickable [Vol X, Pg Y] citations
│   │   │   └── ChatInputDock.tsx           # Prompt input + Spoiler-Guard status pill
│   │   ├── reader/
│   │   │   ├── PaginatedReaderCanvas.tsx   # Resizable right pane + Vertical/Horizontal mode switch
│   │   │   ├── ReaderToolbar.tsx           # Chapter jumper, translation pill, spoiler lock, view mode
│   │   │   ├── EditorialPageCard.tsx       # Individual PDF-style page card with sentence spans
│   │   │   ├── SourcePeekPopover.tsx       # Click-to-peek raw CJK sentence + glossary breakdown
│   │   │   └── SelectionActionPill.tsx     # Floating [Ask AI | Explain | Re-translate | +Glossary]
│   │   ├── audio/
│   │   │   ├── NaturalReaderAudioBar.tsx   # Bottom docked player with speed/voice controls
│   │   │   └── AddTtsEngineModal.tsx       # Pluggable audio engine registration dialog
│   │   └── modals/
│   │       ├── ThemeStudioDrawer.tsx       # Presets + Custom CSS hex pickers + JSON import/export
│   │       └── ExportVolumeModal.tsx       # Export as .epub, .pdf, or .txt (Translated/Bilingual)
│   ├── stores/
│   │   ├── useWorkspaceStore.ts            # Active series_id, split-pane state, WebSocket connection
│   │   ├── useReaderStore.ts               # Active volume_id, page_num, scroll/flip mode, spoiler coord
│   │   └── useAudioStore.ts                # Playing state, active coord_id highlight, audio ring buffer
│   └── services/
│       ├── apiClient.ts                    # REST calls to http://localhost:8080/api/v1/*
│       └── wsManager.ts                    # Multiplexed WebSocket handler for ws://localhost:8080/ws/events
```

---

## 5. Keyboard Shortcuts Contract (Hyprland / Desktop Workflow)
* `Ctrl + K`: Open Global Series & Scene Search palette.
* `Ctrl + B`: Toggle Left Telegram Sidebar (Expanded / Icon-only).
* `Ctrl + \`: Toggle Split-Screen Chat pane while reading a book.
* `Left / Right` or `J / K`: Previous / Next Page Card in the Reader.
* `Space`: Play / Pause NaturalReader TTS Audio Bar (when chat input is not focused).
* `[` / `]`: Skip to Previous / Next Sentence during TTS playback.
