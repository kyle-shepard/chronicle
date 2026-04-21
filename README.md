# Chronicle

### Personal Health Record Intelligence

[![Tauri](https://img.shields.io/badge/Tauri-2.0-blue?logo=tauri)](https://tauri.app)
[![Rust](https://img.shields.io/badge/Rust-1.70+-orange?logo=rust)](https://rust-lang.org)
[![React](https://img.shields.io/badge/React-18-61DAFB?logo=react)](https://react.dev)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

**A local-first, privacy-by-architecture personal health record system. Ingest medical PDFs, extract structured data, track changes over time, and surface correlations across your full health history.**

> ⚠️ **Project Status: Architecture & Design Phase**
>
> The architecture is defined and documented. The codebase in this repository reflects an earlier iteration (PAL V1) and **has not yet been updated** to match the current Chronicle architecture described below. Implementation is underway — see [Project History](#project-history) for how we got here.

---

## What Chronicle Is

Chronicle is a **local-first personal health record intelligence application**. It ingests medical PDFs from patient portals, extracts structured data with confidence scoring, tracks changes over time, and surfaces correlations across your full health history.

It is explicitly a **resource assistant for better doctor conversations** — not a diagnostic tool, not a clinical system.

The core insight: your health data is scattered across patient portals, PDFs, and fragmented memories of what your doctor said. Chronicle gives you a single, structured, searchable record that gets smarter over time.

### Key Differentiators

- **Privacy by architecture** — All data stays on your machine. No cloud, no API keys for core functionality. Your medical records never leave your hardware.
- **Document-first intake** — PDFs from patient portals are the primary data source, not manual conversation entry.
- **Confidence-gated extraction** — Three-tier system: auto-store (high confidence), flag for review (medium), ask the user (low). Medical data accuracy matters.
- **Temporal intelligence** — `occurred_at` vs. `created_at` split. When a lab was drawn matters as much as when you imported the PDF.
- **Cross-record correlation** — Connects patterns across lab results, medications, symptoms, and provider notes over time.

---

## Architecture

Chronicle inherits its architecture from PAL V3, focused on the health record vertical.

```
┌─────────────────────────────────────────────────────────────┐
│                   FRONTEND (React + Tauri)                  │
│  • Health dashboard (labs, meds, timeline)                  │
│  • Document upload & review interface                       │
│  • Chat interface for queries                               │
└─────────────────────────┬───────────────────────────────────┘
                          │ Tauri IPC
┌─────────────────────────▼───────────────────────────────────┐
│                  RUST BACKEND (Tauri 2)                     │
│  • Direct Ollama REST API calls (no framework)              │
│  • PDF ingestion pipeline                                   │
│  • Confidence scoring & extraction                          │
│  • Core memory management (MemGPT-inspired)                 │
│  • Correlation engine                                       │
│  • Reflection system                                        │
└──────────┬──────────────────────────────┬───────────────────┘
           │                              │
┌──────────▼──────────┐    ┌──────────────▼───────────────────┐
│   SQLite + sqlite-vec│    │         Ollama (Local LLM)      │
│  • Events table      │    │  • Qwen 2.5 32B (Q6_K)          │
│  • Documents table   │    │    Synthesis, extraction,        │
│  • Reflections table │    │    conversation, correlation     │
│  • Vector embeddings │    │  • nomic-embed-text              │
│  (single store)      │    │    Event embeddings              │
└──────────────────────┘    └─────────────────────────────────┘
```

### Core Principles

| Principle | What It Means |
|---|---|
| **Local-First** | Data lives on your machine. Privacy by architecture. |
| **One Model, One Call** | Qwen 32B handles classification, extraction, conversation, and synthesis in single calls. No multi-stage routing pipeline. |
| **Self-Managing Memory** | Inspired by MemGPT/Letta. The LLM manages its own context — deciding what to keep, archive, or retrieve. |
| **Events Are First-Class** | Every piece of data is a timestamped event with domain and type tags. Single source of truth. |
| **Structured, Not Flat** | Events have typed fields (not generic text blobs). A lab result has values, units, reference ranges. |
| **Documents as Source of Truth** | PDFs retain full provenance. Every extracted value traces back to a specific page of a specific document. |
| **Honest About Confidence** | Extracted data carries confidence scores. Uncertain values are flagged, never silently stored as fact. |
| **No Frameworks** | Direct Ollama REST API from Rust. No LangChain, no LangGraph, no abstraction layers. |

---

## Data Model

### Events Table (Single Store)

All structured data lives in one table with typed JSON fields per event type:

```sql
CREATE TABLE events (
    event_id    TEXT PRIMARY KEY,
    domain      TEXT NOT NULL,        -- 'health'
    event_type  TEXT NOT NULL,        -- 'lab_result' | 'medication' | 'symptom' | 'vitals' | ...
    summary     TEXT NOT NULL,        -- Human-readable summary
    raw_input   TEXT,                 -- Original source text
    fields      TEXT NOT NULL,        -- JSON: structured domain-specific data
    source      TEXT NOT NULL,        -- 'document' | 'user' | 'system'
    created_at  TEXT NOT NULL,        -- When imported
    occurred_at TEXT NOT NULL,        -- When it actually happened
    embedding   BLOB,                -- Vector embedding (sqlite-vec)
    parent_id   TEXT,                 -- Links related events
    FOREIGN KEY (parent_id) REFERENCES events(event_id)
);
```

### Documents Table (PDF Provenance)

```sql
CREATE TABLE documents (
    document_id TEXT PRIMARY KEY,
    filename    TEXT NOT NULL,
    file_hash   TEXT NOT NULL,        -- Deduplication
    source      TEXT,                 -- 'mychart' | 'quest' | 'manual_upload'
    doc_date    TEXT,                 -- Date on the document
    imported_at TEXT NOT NULL,
    page_count  INTEGER,
    raw_text    TEXT                  -- Full extracted text
);
```

### Core Memory Blocks (Always In Context)

Small editable text blocks (~2-3K tokens) injected into every LLM call:

| Block | Contents |
|---|---|
| `patient_profile` | Demographics, key medical facts, provider list |
| `health_status` | Active conditions, current medications, recent symptoms |
| `recent_results` | Latest lab trends, flagged abnormals |
| `active_concerns` | Items to discuss with provider, pending follow-ups |

---

## Document Ingestion Pipeline

```
PDF uploaded
    │
    ▼
┌─────────────────────────────────────────┐
│  Text extraction (page-by-page)         │
│  Document stored with hash for dedup    │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  Qwen 32B — Structured Extraction       │
│                                         │
│  For each identified data point:        │
│  • Extract structured fields            │
│  • Assign confidence score              │
│  • Link to source document + page       │
│                                         │
│  Confidence gate:                       │
│  • HIGH   → auto-store                  │
│  • MEDIUM → flag for user review        │
│  • LOW    → ask user before storing     │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  Events created with occurred_at        │
│  timestamps from the document, NOT      │
│  the import date                        │
│  Embeddings computed on summaries       │
│  Correlation check against existing     │
│  health events                          │
└─────────────────────────────────────────┘
```

---

## Technology Stack

| Layer | Choice | Rationale |
|---|---|---|
| **App Framework** | Tauri v2 (Rust + React) | Native Windows, single binary, low memory |
| **Core Language** | Rust | Performance, safety, direct Ollama integration |
| **Frontend** | React | Dashboard + document review UI |
| **LLM Runtime** | Ollama | Local model serving, GPU acceleration |
| **LLM Model** | Qwen 2.5 32B (Q6_K) | ~26GB VRAM, handles all tasks in single calls |
| **Embedding Model** | nomic-embed-text | 768 dimensions, 8K context via Ollama |
| **Data Store** | SQLite + sqlite-vec | Events, documents, embeddings — single database |
| **Framework** | None | Direct Ollama REST API from Rust |

### Target Hardware

- NVIDIA RTX 4090 (32GB VRAM)
- Windows 11

---

## Project Phases

### Phase 1: Foundation
- [ ] Tauri v2 project scaffold (replacing existing V1 codebase)
- [ ] SQLite event store + documents table
- [ ] Ollama integration (Qwen 32B + nomic-embed-text)
- [ ] Core memory block system
- [ ] Basic PDF ingestion → structured extraction → event storage
- [ ] Health dashboard v1

### Phase 2: Intelligence
- [ ] Confidence-gated extraction (three-tier)
- [ ] Semantic search over health events (sqlite-vec)
- [ ] Cross-document correlation engine
- [ ] Medication tracking with timeline
- [ ] Lab result trending and flagging

### Phase 3: Reflection & Patterns
- [ ] Nightly reflection system
- [ ] Core memory auto-management
- [ ] Pattern detection (symptom ↔ medication, lab trends)
- [ ] Provider visit preparation summaries

### Phase 4: Polish
- [ ] Multi-provider document handling
- [ ] Data export for provider sharing
- [ ] Performance optimization
- [ ] Backup/restore

---

## Project History

This project has gone through several architectural iterations:

1. **J.A.R.V.I.S. / PAL V1** — Broad personal assistant with 7 domain services, Node.js backend, LanceDB, self-evolution system, voice interface. The code currently in this repo is from this era.
2. **PAL V2** — Architectural refinements. Removed 3B classifier model, added semantic router, refined enrichment pipeline.
3. **PAL V3** — Full architectural reset. Collapsed 9 tables to 1 events table, eliminated LanceDB (single SQLite store), removed semantic router and enrichment queue, upgraded to Qwen 32B, added MemGPT-inspired memory and nightly reflection. Rust backend replacing Node.js. See `docs/PAL_PROJECT_VISION_V3.md`.
4. **Chronicle** *(current)* — Focused the PAL V3 architecture on the health record vertical. Added document ingestion pipeline, confidence scoring, `occurred_at` timestamps, and documents table. Removed calendar, email, TODO, and other broad assistant features from scope.

The existing code in this repository is from iteration 1 and does not reflect the current architecture. A full codebase replacement is planned as part of Phase 1.

### Architectural Documents

| Document | Status |
|---|---|
| `docs/PAL_PROJECT_VISION_V3.md` | Superseded by Chronicle — retained as architectural reference |
| `docs/CHRONICLE_VISION_V1.md` | Current authoritative architecture *(pending)* |
| `docs/MODULES.md` | From V1 — outdated |
| `docs/CONTEXT.md` | From V1 — outdated |
| `docs/ROADMAP.md` | From V1 — outdated |

---

## What Was Removed (From Original J.A.R.V.I.S.)

| Removed | Reason |
|---|---|
| Voice interface (wake word, STT, TTS) | Cut from scope. Can revisit post-MVP. |
| Self-evolution system | Over-engineered for a single-user health tool. |
| 7 domain services (finance, tasks, goals, etc.) | Scope narrowed to health records. |
| Node.js + Express backend | Replaced by Rust for performance and Ollama integration. |
| LanceDB (separate vector store) | Replaced by sqlite-vec. One store, not two. |
| Background enrichment queue | Extraction happens inline during document import. |
| Semantic router | One 32B call handles classification. |
| Multi-turn 24-hour conversation window | Replaced by core memory blocks (always in context). |

---

## License

MIT License — see [LICENSE](LICENSE) for details.

---

**Your medical records. Your machine. Your understanding.**
