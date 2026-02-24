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
 │ RAG INGESTION  │                                   │ RAG Retrieval (v1.0.8c) │
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
| `RAG INGESTION.json` | v1.0.8c | Main ingestion pipeline — extracts, chunks, embeds, and stores documents |
| `RAG Retrieval Sub-Workflow.json` | v1.0.8c | Agentic retrieval with dynamic hybrid search and context expansion |
| `Knowledge Graph Workflow (LightRAG).json` | v1.1 | Insert/update/delete documents in LightRAG knowledge graph |
| `Multimodal RAG Ingestion Sub-workflow.json` | v1.2 | OCR via Mistral, image extraction to Supabase, enriched markdown output |
| `Zep Update Long-Term Memories Sub-workflow.json` | — | Persist conversation messages to Zep threads for long-term memory |

## Ingestion Pipeline

### Supported File Types

| Category | Formats |
|----------|---------|
| Text | Plain Text (.txt), Markdown (.md) |
| Documents | Google Docs, PDF, HTML |
| Spreadsheets | Excel (.xlsx), CSV, Google Sheets |
| Via LlamaParse | Word (.doc/.docx), PowerPoint, RTF, EPUB, Images, Audio, Video |

### Key Features

- **Smart Chunking** - Hierarchical markdown-aware splitting (min 400 / target 600 / max 800 chars) with page number tracking
- **Deduplication** - SHA256 content hashing via `record_manager_v2`; skips unchanged docs, re-indexes changed ones
- **LLM Enrichment** - Claude Haiku 4.5 extracts headline, summary, and custom metadata per document
- **Batch Embedding** - Chunks batched (200/batch) with OpenAI `text-embedding-3-small`, inserted via PostgreSQL `UNNEST`
- **Tabular Data** - Excel/CSV rows stored individually as JSON in `tabular_document_rows`, optionally embedded

### Optional Features (Toggle via Set Data node)

| Flag | Purpose |
|------|---------|
| `contextual_embedding_enabled` | Claude Haiku 4.5 generates contextual snippets per chunk before embedding |
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

**Cohere Reranking** — Results reranked by relevance using Cohere rerank-v3.5

Optional (disabled): Knowledge Graph queries, tabular SQL queries, Zep long-term memory

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

- **OpenAI** — Embeddings (`text-embedding-3-small`), GPT-5.2 (retrieval agent)
- **Anthropic** — Claude Sonnet 4.5 (document enrichment), Claude Opus 4.5 (metadata prep/reranking)
- **Mistral AI** — OCR (`mistral-ocr-latest`)
- **Supabase / PostgreSQL** — Vector store, record management, image storage, edge functions (hybrid search, context expansion)
- **Google Drive** — File triggers and operations
- **LightRAG** — Knowledge graph (optional)
- **Zep** — Long-term memory (optional)
- **LlamaParse** — Advanced document parsing for Word, PowerPoint, RTF, EPUB, and other complex formats
- **Cohere** — Reranking (rerank-v3.5, active)
- **Firecrawl** — Web scraping (disabled by default)

---

## Changelog

### v1.0.8c (5) - 2026-02-24

**LlamaParse Integration (re-enabled):**
- **Fixed LlamaParse upload endpoint** — changed from `/api/v1/files/` (file management API) to `/api/parsing/upload` (parsing API) which uploads and starts the parsing job in one step
- **Fixed upload form data** — changed parameter type from `formData` to `formBinaryData` and field name to `file` so the actual binary file is sent instead of a string literal
- **Re-enabled full LlamaParse polling chain** — enabled all 7 nodes: Upload, Wait1, Set counter, Is Job Ready?, Wait, Get Processing Status, Counter, Get parsed document
- **Word documents (.doc/.docx) now supported** — routed via LlamaParse through the "Other doc types" branch in Switch1

**Model Changes:**
- **Removed duplicate contextual embedding LLM chain** — `Basic LLM Chain1` was running the same prompt as `Basic LLM Chain`, doubling API calls and token usage per chunk. Removed the duplicate
- **Contextual embedding model remains OpenAI** — `Basic LLM Chain` still uses `OpenAI Chat Model1` (gpt-5-nano) for per-chunk contextual descriptions

**Bug Fixes:**
- **Fixed If3 type validation error** — changed condition from `$json` (object notEmpty, strict) to `$json.metadata_name` (string notEmpty, loose). The Supabase metadata query result was failing strict object type validation when receiving empty data
- **Set workflow execution order to v1** — explicit execution order setting

### v1.0.8c - 2026-02-19

**Retrieval Workflow:**
- **Fixed sub-workflow trigger** — changed "When Executed by Another Workflow" from `inputSource: passthrough` to explicitly defined inputs (query, type, session_id, dense/sparse/ilike/fuzzy weights, fuzzy_threshold). Passthrough mode silently fails when called from an agent toolWorkflow — defined inputs are required for `$fromAI()` parameter binding
- **Removed stale pinned data** — cleared pinned bitcoin.pdf test results from "Trigger Dynamic Hybrid Search" that were overriding live Supabase calls
- **Fixed Cohere reranking code** — rewrote "Return Reordered Items1" to handle HTTP Request node response format (single item containing array vs multiple items). Re-enabled full reranking pipeline (If3, Create Array, Rerank with Cohere 3.5, Return Reordered Items1)
- **Removed self-referencing Dynamic Hybrid Search3** — eliminated the self-calling tool node and `callerPolicy: any` setting; "Dynamic Hybrid Search" (pointing to Blueprint workflow) now serves as the sole search tool
- **Switched retrieval agent to GPT-5.2** — replaced Claude Opus 4.6 with GPT-5.2 for Agentic RAG node
- **Switched Prep Metadata1 to Claude Opus 4.5** — replaced GPT-5.2 with Anthropic Claude Opus 4.5 for metadata prep/reranking
- **Removed memory context window limit** — `contextWindowLength` set to null on Supabase Short-Term Memory1 for full conversation history
- **Simplified agent system prompt** — removed strict citation/grounding rules in favor of standard response format

**Ingestion Workflow:**
- **Switched enrichment LLM to Claude Sonnet 4.5** — replaced OpenAI (GPT-4.1-mini) with Anthropic Claude Sonnet 4.5 for document headline/summary/metadata extraction
- **Disabled contextual embeddings pipeline** — turned off 8 nodes (contextual embedding switch, LLM chain, both models, Edit Fields, Wait 5s, rate limiting note) to reduce latency and cost
- **Updated Mistral OCR credentials** — switched from general "Mistral Cloud Free Account" to dedicated "Mistral OCR" credential across all 3 OCR nodes
- Synced both workflows from live n8n exports

### v0.1.8b - 2025-02-16
- **Added Anthropic Claude Sonnet 4.5** model nodes to both retrieval agent and ingestion pipeline
- **Migrated LlamaParse to v1 API** — new endpoint, Bearer auth, simplified upload params
- **Added Edit Fields SET node** in ingestion for data minimization
- Explicit credential bindings on all Anthropic model nodes
- Synced both workflows from live n8n exports

### v0.1.8 - 2025-02-16
- **Enabled OCR** — `ocr_enabled` flag set to `true` for optical character recognition on ingested documents
- **Enabled contextual embeddings** — `contextual_embedding_enabled` flag set to `true` for richer, context-aware vector embeddings
- **Cohere reranking working** — Cohere API credential configured and named; reranking pipeline ready to be re-enabled
- Synced ingestion workflow from live n8n export

### v0.1.7 - 2025-02-15
- **Fixed public chat URL** — set chat trigger response mode to `lastNode` to resolve "No response received" streaming error
- **Multi-language support** — replaced vague language instruction with explicit rule: respond in the same language the user writes in
- **Recycling Bin check interval** — reduced from 10 min to 2 min for faster document deletion
- Chat greeting now shows version number (`v0.1.7`)
- Added **User Manual** (`USER MANUAL.md`) for end users
- Corrected **supported file types** — Word, PowerPoint, RTF, EPUB, images, audio, video require LlamaParse (disabled)
- Synced both workflows from live n8n exports

### v0.1.6 - 2025-02-15
- **Fixed Dynamic Hybrid Search** — replaced broken self-referencing tool node (had invalid workflow ID `R_I-MybamT9nBjj0P-Gg3/29b9d5` and malformed expressions) with new `Dynamic Hybrid Search3` node using correct ID and clean `$fromAI()` mappings
- Set retrieval workflow **callerPolicy to `any`** so sub-workflow calls are not blocked
- Updated credential references to match live n8n instance (Mistral, Google Drive, Supabase)
- Synced both workflows from live n8n exports

### v0.1.5.2 - 2025-02-15
- Enabled `send_tabular_data_to_vector_store = true` so Excel/CSV/Sheets data gets embedded into the vector store for retrieval

### v0.1.5.1 - 2025-02-15
- Reverted **Cohere reranking** pipeline back to disabled (caused infinite hang — Cohere API key was missing; now resolved in v0.1.8)

### v0.1.5 - 2025-02-15
- **Anti-hallucination improvements** to RAG Retrieval Sub-Workflow:
  - ~~Enabled Cohere rerank-v3.5 pipeline~~ (reverted in v0.1.5.1)
  - Set agent model (GPT-5.2) **temperature to 0.1** for more factual responses
  - Strengthened **system prompt** with inline citation rules, grounding requirements, and partial-answer handling

### v0.1.4 - 2025-02-15
- Enabled **CSV** file support (Extract from CSV node)
- Enabled **Google Sheets** support (Get google sheet info + Get row(s) in sheet) with OAuth2 credentials
- Enabled **Webpage Markdown** processing node

### v0.1.3 - 2025-02-15
- Added **RAG Retrieval Sub-Workflow** (v2.3.3) — agentic retrieval with dynamic hybrid search
- Added **Knowledge Graph Workflow** (LightRAG v1.1) — knowledge graph sub-workflow
- Added **Multimodal RAG Ingestion Sub-workflow** (v1.2) — OCR and image processing
- Added **Zep Update Long-Term Memories Sub-workflow** — conversation persistence
- Updated README to document full system architecture

### v0.1.2 - 2025-02-15
- Enabled **Google Docs** support (Get a document node) with OAuth2 credentials
- Added credential bindings for LlamaParse nodes (Get Processing Status, Get parsed document)
- Preserved existing Supabase credential on Update our Record Manager

### v0.1.1 - 2025-02-15
- Enabled **Plain Text** file support (Extract from text file node)
- Enabled **Excel** file support with full pipeline:
  - Extract from Excel -> Aggregate3 -> Array keys + Summarize -> Merge -> Set Text for Tabular Data

### v0.1.0 - Initial
- Core RAG ingestion pipeline with Google Drive integration
- Support for Markdown, Google Docs, PDF, HTML, CSV, Google Sheets, and other document types
- Hierarchical markdown-aware chunking with configurable sizes
- OpenAI embedding generation with batch processing
- PostgreSQL pgvector storage with record manager deduplication
- LLM-based document enrichment (headline, summary, metadata)
- Optional: contextual embeddings, LightRAG, multimodal RAG, OCR, LlamaParse
