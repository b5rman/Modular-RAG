# Modular RAG

A production-grade n8n RAG (Retrieval-Augmented Generation) system composed of 5 workflows: a main ingestion pipeline, a retrieval agent, and 3 specialized sub-workflows for knowledge graphs, multimodal content, and long-term memory.

## System Architecture

```
                    INGESTION                                    RETRIEVAL
                    ─────────                                    ─────────

 Google Drive / Webhook                                  Chat UI / Parent Workflow
         │                                                        │
         ▼                                                        ▼
 ┌────────────────┐                                   ┌─────────────────────────┐
 │ RAG INGESTION  │                                   │ RAG Retrieval (v2.3.3)  │
 │                │                                   │                         │
 │ Extract → Chunk│                                   │ Agent (GPT-5.2)         │
 │ → Embed → Store│                                   │ ├─ Dynamic Hybrid Search│
 │                │──┐                                │ ├─ Context Expansion    │
 └────────────────┘  │                                │ ├─ Doc Hierarchy        │
                     │                                │ └─ (Knowledge Graph)    │
        ┌────────────┼───────────┐                    └─────────────┬───────────┘
        ▼            ▼           ▼                                  ▼
 ┌────────────┐ ┌──────────┐                             ┌──────────────────┐
 │ Multimodal │ │ Knowledge│    ┌───────────────┐        │ Zep Long-Term    │
 │ RAG (v1.2) │ │ Graph    │    │ documents_v2  │        │ Memories         │
 │            │ │ (v1.1)   │    │ (pgvector)    │◄───────│                  │
 │ Mistral OCR│ │ LightRAG │    └───────────────┘        │ Persists chat    │
 │ → Supabase │ │          │                             │ context to Zep   │
 └────────────┘ └──────────┘                             └──────────────────┘
```

## Workflows

| File | Version | Purpose |
|------|---------|---------|
| `RAG INGESTION.json` | v1.2 | Main ingestion pipeline — extracts, chunks, embeds, and stores documents |
| `RAG Retrieval Sub-Workflow.json` | v2.3.3 | Agentic retrieval with dynamic hybrid search and context expansion |
| `Knowledge Graph Workflow (LightRAG).json` | v1.1 | Insert/update/delete documents in LightRAG knowledge graph |
| `Multimodal RAG Ingestion Sub-workflow.json` | v1.2 | OCR via Mistral, image extraction to Supabase, enriched markdown output |
| `Zep Update Long-Term Memories Sub-workflow.json` | — | Persist conversation messages to Zep threads for long-term memory |

## Ingestion Pipeline

### Supported File Types

| Category | Formats |
|----------|---------|
| Text | Plain Text, Markdown |
| Documents | Google Docs, PDF, HTML, Word (.doc/.docx), PowerPoint, RTF, EPUB, and more |
| Spreadsheets | Excel (.xlsx), CSV, Google Sheets |
| Media | Images (JPEG, PNG, GIF, etc.), Audio (MP3, WAV), Video (MP4, WebM) |

### Key Features

- **Smart Chunking** - Hierarchical markdown-aware splitting (min 400 / target 600 / max 800 chars) with page number tracking
- **Deduplication** - SHA256 content hashing via `record_manager_v2`; skips unchanged docs, re-indexes changed ones
- **LLM Enrichment** - GPT-4 mini extracts headline, summary, and custom metadata per document
- **Batch Embedding** - Chunks batched (200/batch) with OpenAI `text-embedding-3-small`, inserted via PostgreSQL `UNNEST`
- **Tabular Data** - Excel/CSV rows stored individually as JSON in `tabular_document_rows`, optionally embedded

### Optional Features (Toggle via Set Data node)

| Flag | Purpose |
|------|---------|
| `contextual_embedding_enabled` | GPT-4 mini generates contextual snippets per chunk before embedding |
| `lightrag_enabled` | Routes to LightRAG sub-workflow for knowledge graph construction |
| `multimodal_rag_enabled` | Routes to multimodal sub-workflow for image/visual content |
| `ocr_enabled` | Mistral AI OCR for scanned documents and images |
| `send_tabular_data_to_vector_store` | Also embed tabular rows into the vector store |

### Triggers

- **Google Drive - New Files** - Monitors "Rag Files" folder, polls every 1 min
- **Google Drive - Updated Files** - Same folder, polls every 1 min
- **Webhook** - HTTP POST endpoint for programmatic ingestion

## Retrieval Agent

The retrieval workflow uses an agentic RAG pattern with GPT-5.2 and 3 tools:

**Dynamic Hybrid Search** — 4-component search with configurable weights:

| Component | Default | Method |
|-----------|---------|--------|
| Dense | 0.5 | Vector similarity (text-embedding-3-small) |
| Sparse | 0.5 | BM25 lexical search |
| ilike | 0 | Wildcard pattern matching |
| Fuzzy | 0 | Postgres fuzzy matching (threshold 0.8) |

**Context Expansion** — Fetches neighboring/parent chunks via Supabase edge function

**Fetch Document Hierarchy** — Retrieves document structure from record_manager_v2

Optional (disabled): Knowledge Graph queries, tabular SQL queries, Cohere reranking, Zep long-term memory

## Sub-Workflows

### Knowledge Graph (LightRAG v1.1)
Called by the ingestion pipeline when `lightrag_enabled = true`. Compares document hashes to determine action:
- **New document** → Insert into LightRAG, store `graph_id` in record manager
- **Changed document** → Delete old entry, re-insert
- **Unchanged** → Skip

### Multimodal RAG (v1.2)
Called by the ingestion pipeline when `multimodal_rag_enabled = true`:
1. Uploads file to Mistral OCR (`mistral-ocr-latest`)
2. Extracts pages and images
3. Uploads images to Supabase storage
4. Returns enriched markdown with hosted image URLs and page markers

### Zep Long-Term Memories
Called by the retrieval workflow to persist conversation context:
- Posts messages to Zep thread
- Auto-creates thread if it doesn't exist
- Inputs: `session_id`, `user_id`, `message_content`

## Database Tables

| Table | Purpose |
|-------|---------|
| `record_manager_v2` | Document tracking, deduplication, processing status, graph_id |
| `documents_v2` | Vector embeddings (content, metadata, embedding via pgvector) |
| `tabular_document_rows` | Row-level data from Excel/CSV/Sheets |
| `metadata_fields` | Custom metadata field definitions for LLM enrichment |

## External Services

- **OpenAI** — Embeddings (`text-embedding-3-small`), GPT-5.2 (retrieval agent), GPT-4 mini/4o (enrichment, metadata filtering)
- **Mistral AI** — OCR (`mistral-ocr-latest`)
- **Supabase / PostgreSQL** — Vector store, record management, image storage, edge functions (hybrid search, context expansion)
- **Google Drive** — File triggers and operations
- **LightRAG** — Knowledge graph (optional)
- **Zep** — Long-term memory (optional)
- **LlamaParse** — Advanced document parsing (disabled by default)
- **Cohere** — Reranking (disabled by default)
- **Firecrawl** — Web scraping (disabled by default)

---

## Changelog

### v1.5 - 2025-02-15
- **Anti-hallucination improvements** to RAG Retrieval Sub-Workflow:
  - Enabled **Cohere rerank-v3.5** pipeline (If3 → Create Array → Rerank → Return Reordered Items)
  - Set agent model (GPT-5.2) **temperature to 0.1** for more factual responses
  - Strengthened **system prompt** with inline citation rules, grounding requirements, and partial-answer handling

### v1.4 - 2025-02-15
- Enabled **CSV** file support (Extract from CSV node)
- Enabled **Google Sheets** support (Get google sheet info + Get row(s) in sheet) with OAuth2 credentials
- Enabled **Webpage Markdown** processing node

### v1.3 - 2025-02-15
- Added **RAG Retrieval Sub-Workflow** (v2.3.3) — agentic retrieval with dynamic hybrid search
- Added **Knowledge Graph Workflow** (LightRAG v1.1) — knowledge graph sub-workflow
- Added **Multimodal RAG Ingestion Sub-workflow** (v1.2) — OCR and image processing
- Added **Zep Update Long-Term Memories Sub-workflow** — conversation persistence
- Updated README to document full system architecture

### v1.2 - 2025-02-15
- Enabled **Google Docs** support (Get a document node) with OAuth2 credentials
- Added credential bindings for LlamaParse nodes (Get Processing Status, Get parsed document)
- Preserved existing Supabase credential on Update our Record Manager

### v1.1 - 2025-02-15
- Enabled **Plain Text** file support (Extract from text file node)
- Enabled **Excel** file support with full pipeline:
  - Extract from Excel -> Aggregate3 -> Array keys + Summarize -> Merge -> Set Text for Tabular Data

### v1.0 - Initial
- Core RAG ingestion pipeline with Google Drive integration
- Support for Markdown, Google Docs, PDF, HTML, CSV, Google Sheets, and other document types
- Hierarchical markdown-aware chunking with configurable sizes
- OpenAI embedding generation with batch processing
- PostgreSQL pgvector storage with record manager deduplication
- LLM-based document enrichment (headline, summary, metadata)
- Optional: contextual embeddings, LightRAG, multimodal RAG, OCR, LlamaParse
