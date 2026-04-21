# PAL — Personal AI Liaison

### *"Bringing JARVIS to Life"*

[![Tauri](https://img.shields.io/badge/Tauri-2.0-blue?logo=tauri)](https://tauri.app)
[![Rust](https://img.shields.io/badge/Rust-1.70+-orange?logo=rust)](https://rust-lang.org)
[![React](https://img.shields.io/badge/React-18-61DAFB?logo=react)](https://react.dev)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

**A local-first, Windows-native personal AI assistant with structured memory, cross-domain intelligence, and self-managing context. Privacy by architecture — everything runs on your machine.**

> ⚠️ **Project Status: Architecture & Design Phase (V3)**
>
> PAL has gone through three architectural iterations. The current V3 architecture is fully designed and documented, but the **codebase in this repository still reflects the original V1 implementation** (Node.js backend, 7 domain services, LanceDB). A full codebase replacement is planned. See [Project History](#project-history) for details.

---

## What PAL Is

PAL is a local-first AI assistant that runs persistently — from boot, eventually 24/7 on a dedicated server. It combines a local LLM, structured event storage, self-managing memory, and external system integration to serve as a proactive life-management copilot.

PAL is JARVIS. Not a food tracker that might become JARVIS someday. The architecture reflects the full vision from day one — conversational partner, schedule manager, health tracker, project planner, and pattern detector — even if individual capabilities ship in phases.

**The key insight:** Every existing local AI assistant stores knowledge as flat, unstructured blobs. PAL's domain-aware structured event storage with self-managing memory, cross-domain correlation, and nightly self-reflection is the differentiation. The memory system isn't a feature — it's the product.

### What PAL Is Not

- **Not a chatbot.** Conversation is the interface, not the product. The product is what PAL *knows*, the connections it draws, and the actions it takes.
- **Not cloud-dependent.** Core functions run entirely on your machine. External API calls are explicit integrations, not dependencies.
- **Not a framework showcase.** No LangChain, no LangGraph. Direct Ollama REST API calls from Rust.

---

## Domain Verticals

PAL's architecture is domain-agnostic — the events table, core memory, and reflection system work across any life domain. Development is organized around focused verticals built on the shared foundation.

### Chronicle — Personal Health Record Intelligence *(Active Development)*

The first vertical. Chronicle ingests medical PDFs from patient portals, extracts structured data with confidence scoring, tracks changes over time, and surfaces correlations across your full health history. It is a resource assistant for better doctor conversations — not a diagnostic tool.

**What Chronicle adds to PAL's core:**

- Document ingestion pipeline (PDF → structured extraction → event storage)
- Three-tier confidence gating (auto-store / flag for review / ask user)
- `occurred_at` vs. `created_at` timestamp split for medical data accuracy
- Documents table for PDF provenance tracking
- Health-specific core memory blocks (patient profile, health status, recent results)

See `docs/CHRONICLE_VISION_V1.md` for the full Chronicle architecture.

### Future Verticals *(Planned)*

These are part of PAL's full vision and share the same architectural foundation:

- **Work** — TODO tracking, project status, meeting notes, cross-referencing workload vs. schedule
- **Personal** — Goals, habits, milestones, personal commitments vs. calendar
- **Scheduling** — Calendar awareness across multiple organizations, time-block suggestions, conflict detection
- **Finance** — Expense tracking, bill management, category analytics
- **Email** — Inbox triage, categorization, flagging with user confirmation

---

## Architecture (V3)

```
┌─────────────────────────────────────────────────────────────┐
│                   FRONTEND (React + Tauri)                  │
│  • Dashboard (domain-specific views)                        │
│  • Chat interface with streaming responses                  │
│  • Document upload & review (Chronicle)                     │
└─────────────────────────┬───────────────────────────────────┘
                          │ Tauri IPC
┌─────────────────────────▼───────────────────────────────────┐
│                  RUST BACKEND (Tauri 2)                     │
│  • Direct Ollama REST API calls (no framework)              │
│  • Single-pass intake pipeline                              │
│  • Core memory management (MemGPT-inspired)                 │
│  • Correlation engine                                       │
│  • Nightly reflection system                                │
│  • Domain-specific ingestion (PDF pipeline for Chronicle)   │
└──────────┬──────────────────────────────┬───────────────────┘
           │                              │
┌──────────▼──────────┐    ┌──────────────▼───────────────────┐
│  SQLite + sqlite-vec │    │         Ollama (Local LLM)       │
│  • Events table      │    │  • Qwen 2.5 32B (Q6_K)           │
│  • Documents table   │    │    All tasks: classification,    │
│  • Reflections table │    │    extraction, conversation,     │
│  • Vector embeddings │    │    synthesis, reflection         │
│  (single store)      │    │  • nomic-embed-text              │
└──────────────────────┘    │    Event embeddings              │
                            └──────────────────────────────────┘
```

### Core Principles

| Principle | What It Means |
|---|---|
| **JARVIS First** | The full assistant vision drives every decision. Every system is domain-agnostic from day one. |
| **Local-First** | Data lives on your machine. Privacy by architecture. |
| **One Model, One Call** | Qwen 32B handles classification, extraction, conversation, and synthesis in single calls. No separate classifier. No multi-stage routing. |
| **Self-Managing Memory** | Inspired by MemGPT/Letta. PAL manages its own context — deciding what to keep, archive, or retrieve. |
| **Events Are First-Class** | Every piece of data is a timestamped event with domain and type tags in a single table. |
| **Structured, Not Flat** | Events have typed JSON fields appropriate to their domain. Structure enables precise queries. |
| **Proactive, Not Just Reactive** | PAL surfaces connections and flags patterns without being asked. |

---

## Data Model

### Events Table (Single Source of Truth)

One table for all domains. Event types have domain-specific JSON fields.

```sql
CREATE TABLE events (
    event_id    TEXT PRIMARY KEY,       -- UUID
    domain      TEXT NOT NULL,          -- 'health' | 'work' | 'personal' | 'system'
    event_type  TEXT NOT NULL,          -- 'lab_result' | 'food_log' | 'todo' | 'symptom' | ...
    summary     TEXT NOT NULL,          -- Human-readable summary
    raw_input   TEXT,                   -- Original source text
    fields      TEXT NOT NULL,          -- JSON: structured domain-specific data
    source      TEXT NOT NULL,          -- 'user' | 'document' | 'calendar' | 'system'
    created_at  TEXT NOT NULL,          -- When stored
    occurred_at TEXT NOT NULL,          -- When it actually happened
    embedding   BLOB,                  -- Vector embedding (sqlite-vec)
    parent_id   TEXT,                   -- Links related events
    FOREIGN KEY (parent_id) REFERENCES events(event_id)
);
```

### Core Memory Blocks (Always In Context)

Small editable text blocks (~2-3K tokens) injected into every LLM call. PAL reads and writes these to maintain persistent awareness.

| Block | Contents |
|---|---|
| `user_profile` | Name, preferences, communication style, key facts |
| `health_status` | Active conditions, prescriptions, dietary goals, symptoms |
| `active_projects` | Current projects with status and next steps |
| `today_context` | Today's schedule, pending TODOs, flagged items |
| `recent_patterns` | Cross-domain observations from reflection |
| `learned_behaviors` | Self-improvement notes from nightly reflection |

### Reflection System

PAL reviews each day's events, updates core memory, notes self-corrections, and builds compounding knowledge over time. Learned behaviors require multiple occurrences before becoming permanent (durability gate).

---

## Technology Stack

| Layer | Choice | Rationale |
|---|---|---|
| **App Framework** | Tauri v2 (Rust + React) | Native Windows, single binary, persistent system tray |
| **Core Language** | Rust | Performance, safety, direct Ollama integration |
| **Frontend** | React | Dashboard + chat UI in Tauri webview |
| **LLM Runtime** | Ollama | Local model serving, GPU acceleration |
| **LLM Model** | Qwen 2.5 32B (Q6_K) | ~26GB VRAM, fully resident on RTX 4090 |
| **Embedding Model** | nomic-embed-text | 768 dimensions, 8K context via Ollama |
| **Data Store** | SQLite + sqlite-vec | Events, documents, embeddings — one database |
| **Framework** | None | Direct Ollama REST API from Rust |

### Target Hardware

- NVIDIA RTX 4090 (32GB VRAM)
- Windows 11

---

## Project Phases

### Phase 1: Foundation + Chronicle Core
- [ ] Tauri v2 project scaffold (replacing V1 codebase)
- [ ] SQLite event store + documents table with schema
- [ ] Ollama integration (Qwen 32B + nomic-embed-text)
- [ ] Core memory block system (read/write from LLM)
- [ ] Basic intake pipeline: input → single LLM call → event storage → response
- [ ] PDF ingestion pipeline (Chronicle)
- [ ] Health dashboard v1

### Phase 2: Chronicle Intelligence + First Domains
- [ ] Confidence-gated extraction (three-tier)
- [ ] Semantic search over events (sqlite-vec)
- [ ] Cross-document correlation engine
- [ ] Lab result trending and medication timeline
- [ ] Food logging with inline nutrition lookup
- [ ] TODO tracking with status management
- [ ] Dashboard v2: health + tasks + projects

### Phase 3: Calendar, Email, Cross-Domain Intelligence
- [ ] Calendar integration (read-only, multiple orgs)
- [ ] Morning briefing (proactive, schedule-aware)
- [ ] Email integration (read + categorize + flag/archive)
- [ ] Nightly reflection system
- [ ] Core memory auto-management
- [ ] Pattern detection across all active domains

### Phase 4: Polish + Proactive Intelligence
- [ ] Proactive pattern surfacing
- [ ] Full scenario suite validation
- [ ] Provider visit preparation summaries (Chronicle)
- [ ] Project narrative generation
- [ ] Performance optimization

### Phase 5: Voice Module
- [ ] Wake word detection ("Hey PAL")
- [ ] STT pipeline (faster-whisper + Silero VAD)
- [ ] TTS with streaming from LLM output
- [ ] Voice ↔ text mid-conversation switching
- [ ] Target: ≤3 seconds wake-to-first-audio

### Phase 6: Advanced Intelligence
- [ ] Monthly reflections and long-term trend analysis
- [ ] Cross-domain correlation deepening
- [ ] Data backup/export/restore
- [ ] Server deployment option (24/7 operation)

---

## Personality

JARVIS (Iron Man) + Guppy (Bobiverse). Competent peers. Not servants, not therapists.

> *"Be brief by default. Match the user's energy. Offer observations when you notice patterns, framed as information not advice. You can be dry or witty when appropriate but never at the user's expense. Don't narrate your own reasoning. 'Done' is a complete response when the situation warrants it. Never be sycophantic."*

---

## Project History

This project has gone through several architectural iterations. Each reset was deliberate — driven by simplification, not scope creep.

| Iteration | What Changed |
|---|---|
| **V1 (J.A.R.V.I.S.)** | Node.js + Express backend, 7 domain-specific tables, LanceDB for vectors, self-evolution system, voice interface, qwen2.5:7b. The code currently in this repo. |
| **V2** | Removed 3B classifier model, added semantic router, refined enrichment pipeline. |
| **V3 (current architecture)** | Full reset. Rust backend, single events table with typed JSON, eliminated LanceDB (SQLite + sqlite-vec), removed semantic router and enrichment queue, upgraded to Qwen 32B, added MemGPT-inspired memory and nightly reflection. |
| **Chronicle vertical** | First domain vertical. Focused health record intelligence with document ingestion, confidence scoring, `occurred_at` timestamps, and documents table. Built on V3 foundation — does not replace PAL's broader scope. |

### What Was Removed From V1 (And Why)

| Removed | Reason |
|---|---|
| Node.js + Express backend | Replaced by Rust for performance and direct Ollama integration |
| 7 domain-specific SQL tables | Collapsed into single events table with typed JSON fields |
| LanceDB (separate vector store) | Replaced by sqlite-vec. One store, not two. |
| Self-evolution system | Over-engineered. New event types = new JSON shapes, no schema migration needed. |
| Background enrichment queue | Enrichment happens inline during intake |
| Semantic router | One 32B call handles classification |
| 3B classifier model | Unnecessary with a capable 32B model |
| qwen2.5:7b | Upgraded to Qwen 2.5 32B (Q6_K) on RTX 4090 |

### Architectural Documents

| Document | Status |
|---|---|
| `docs/PAL_PROJECT_VISION_V3.md` | Current authoritative PAL architecture |
| `docs/CHRONICLE_VISION_V1.md` | Chronicle vertical architecture |
| `docs/MODULES.md` | From V1 — outdated |
| `docs/CONTEXT.md` | From V1 — outdated |
| `docs/ROADMAP.md` | From V1 — outdated |
| `CLAUDE.md` | AI development guidelines — needs V3 update |

---

## License

MIT License — see [LICENSE](LICENSE) for details.

---

**Your data. Your machine. Your AI.**
