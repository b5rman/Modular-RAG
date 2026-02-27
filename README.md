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
 │ RAG INGESTION  │                                   │ RAG Retrieval           │
 │                │                                   │                         │
 │ Extract → Chunk│                                   │ Agent (Sonnet 4.6)      │
 │ → Embed → Store│                                   │ ├─ Dynamic Hybrid Search│
 │                │──┐                                │ ├─ Context Expansion    │
 └────────────────┘  │                                │ ├─ Doc Hierarchy        │
                     │                                │ └─ (Knowledge Graph)    │
        ┌────────────┼───────────┐                    └─────────────┬───────────┘
        ▼            ▼           ▼                                  ▼
 ┌────────────┐ ┌──────────┐                             ┌──────────────────┐
 │ Multimodal │ │ Knowledge│    ┌───────────────┐        │ Zep Long-Term    │
 │ RAG        │ │ Graph    │    │ documents_v2  │        │ Memories         │
 │            │ │          │    │ (pgvector)    │◄───────│                  │
 │ Mistral OCR│ │ LightRAG │    └───────────────┘        │ Persists chat    │
 │ → Supabase │ │          │                             │ context to Zep   │
 └────────────┘ └──────────┘                             └──────────────────┘
```

## Workflows

| File | Nodes | Purpose |
|------|-------|---------|
| `RAG INGESTION v0.2.2.json` | 144 | Main ingestion pipeline — extracts, chunks, embeds, and stores documents |
| `RAG Retrieval Sub-Workflow v0.2.2.json` | 65 | Agentic retrieval with dynamic hybrid search, Voyage AI reranking, and citation verification |

> **Sub-workflows** (Knowledge Graph/LightRAG, Multimodal RAG, Zep Long-Term Memory) are configured in n8n but removed from GitHub — they are not active in production.

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
- **Cache Warmup** - Pre-creates Anthropic ephemeral cache once per document before the chunk loop, ensuring 100% prompt cache reads on all contextual embedding requests
- **Retry Mechanism** - Rate limit errors on contextual embedding trigger a 60s wait + retry, with fallback to regular embedding if retry also fails
- **OOM Protection** - Document text truncated to 100K chars for contextual embedding context, preventing n8n worker memory exhaustion on large PDFs (200+ pages)
- **Tabular Data** - Excel/CSV rows stored individually as JSON in `tabular_document_rows`, optionally embedded

### Optional Features (Toggle via Set Data node)

| Flag | Purpose |
|------|---------|
| `contextual_embedding_enabled` | Claude Haiku 4.5 generates contextual snippets per chunk before embedding (with Anthropic prompt caching for cost reduction) |
| `lightrag_enabled` | Routes to LightRAG sub-workflow for knowledge graph construction |
| `multimodal_rag_enabled` | Routes to multimodal sub-workflow for image/visual content |
| `ocr_enabled` | Mistral AI OCR for scanned documents and images |
| `send_tabular_data_to_vector_store` | Also embed tabular rows into the vector store |

### Triggers

- **Google Drive - New Files** - Monitors "Rag Files" folder, polls every 1 min
- **Google Drive - Updated Files** - Same folder, polls every 1 min
- **Webhook** - HTTP POST endpoint for programmatic ingestion

## Retrieval Agent

The retrieval workflow uses an agentic RAG pattern with Claude Sonnet 4.6 (extended thinking) and 3 tools:

**Dynamic Hybrid Search** — 4-component search with configurable weights:

| Component | Default | Method |
|-----------|---------|--------|
| Dense | 0.5 | Vector similarity (text-embedding-3-small) |
| Sparse | 0.5 | BM25 lexical search |
| ilike | 0 | Wildcard pattern matching |
| Fuzzy | 0 | Postgres fuzzy matching (threshold 0.8) |

**Context Expansion** — Fetches neighboring/parent chunks via Supabase edge function

**Fetch Document Hierarchy** — Retrieves document structure from record_manager_v2

**Voyage AI Reranking** — Results reranked by relevance using Voyage AI rerank-2.5 (top_k: 20)

**Citation Verification** — Post-processing Code node that splits the agent's output on a `---SOURCES_JSON---` delimiter, validates structured citation objects (doc_name, doc_id, pages, chunk_indices), and builds a clean References section. Graceful fallback if no sources are provided.

Optional (disabled): Knowledge Graph queries, Zep long-term memory

## Sub-Workflows

### Knowledge Graph (LightRAG)
Called by the ingestion pipeline when `lightrag_enabled = true`. Compares document hashes to determine action:
- **New document** → Insert into LightRAG, store `graph_id` in record manager
- **Changed document** → Delete old entry, re-insert
- **Unchanged** → Skip

### Multimodal RAG
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

- **OpenAI** — Embeddings (`text-embedding-3-small`)
- **Anthropic** — Claude Sonnet 4.6 (retrieval agent with extended thinking), Claude Haiku 4.5 (document enrichment, contextual embeddings with prompt caching), Claude Opus 4.5 (metadata prep/reranking)
- **Mistral AI** — OCR (`mistral-ocr-latest`)
- **Supabase / PostgreSQL** — Vector store, record management, image storage, edge functions (hybrid search, context expansion)
- **Google Drive** — File triggers and operations
- **LightRAG** — Knowledge graph (optional)
- **Zep** — Long-term memory (optional)
- **LlamaParse** — Advanced document parsing for Word, PowerPoint, RTF, EPUB, and other complex formats
- **Voyage AI** — Reranking (rerank-2.5)
- **Firecrawl** — Web scraping (disabled by default)

---

## Changelog

### v0.2.2 - 2026-02-27

**Ingestion Pipeline Reliability & Caching:**
- **Added Cache Warmup node** — single HTTP Request that pre-creates the Anthropic ephemeral cache (once per document) before the chunk loop starts. All chunks read from the warm cache, eliminating the double-write on chunk 0. Zero net cost increase (~2-3s overhead per document)
- **Added Restore Chunks node** — Code node between Cache Warmup and Split Out1 that restores the chunks data from Merger via `$('Merger').first().json`, since HTTP Request nodes replace their input data with the API response
- **Added contextual embedding retry mechanism** — rate limit errors trigger a 60s wait + automatic retry, with fallback to regular embedding if the retry also fails. Handles Anthropic's 450K input tokens/minute limit on large documents
- **OOM protection** — document text truncated to 100K chars (`.substring(0, 100000)`) at the `Set up Text for Embedding` source node, preventing n8n Cloud worker memory exhaustion on 200+ page PDFs
- **Prompt caching regression fix** — restored `batchInterval: 500` on contextual embedding HTTP Request nodes after typeVersion hardening (4.2→4.4) silently stripped it
- **Loop over Chunks batchSize increased** — 25 → 50 for reduced loop overhead (independent of HTTP Request batching which handles prompt cache pacing)
- **Ingestion pipeline now 144 nodes** (was 140 in v0.2.0)

**Answer Consistency Tuning (Retrieval):**
- **Temperature reduced to 0.2** (was 0.7 default) on Claude Sonnet 4.6 for more deterministic factual answers
- **Extended thinking properly configured** — `thinkingBudget: 10000` with `maxTokensToSample: 16000` (was missing, causing budget validation errors)
- **SOP updated** — mandatory minimum 2 searches with different query formulations
- **Voyage AI rerank top_k increased** — 15 → 20 for broader context coverage
- **Citation sections field added** — references now include specific section headings from `cascading_path` metadata instead of meaningless `pages: [1]`

**Temperature Optimization (all workflows):**
- **Metadata prep LLMs set to 0.4** — Anthropic Chat Model1 (Haiku 4.5, ingestion) and Anthropic Chat Model (Haiku 4.5, retrieval Prep Metadata1). Was 0.7 default
- **Mistral Cloud Chat Model set to 0.4** — fallback enrichment node (Enrich1). Was 0.7 default
- **Contextual Embedding API set to 0.3** — lower variance for short factual summaries. Was 1.0 (Anthropic API default)
- **Cache Warmup set to 0** — output is irrelevant, only triggers cache creation. Was 1.0 default
- **Contextual embedding prompt enhanced** — added rule to include key entities, technical terms, and relationships in summaries to improve search recall

### v0.2.1 - 2026-02-26

**Citation Verification System (Phase 1):**
- **Added `Format & Verify Citations` code node** — post-processes agent output by splitting on `---SOURCES_JSON---` delimiter, validating citation structure (doc_name, doc_id, pages, chunk_indices, relevance), and building a clean References section. Invalid citations are logged as warnings rather than silently dropped
- **Updated agent system prompt** — new structured output format requiring a `---SOURCES_JSON---` delimiter between the answer and a JSON array of source objects. Every factual claim must trace to at least one source with exact metadata from the retrieved chunks
- **Switched chat trigger to `lastNode` response mode** — required for citation post-processing (no streaming; users wait for full response)
- **Connection**: `Agentic RAG 1` → `Format & Verify Citations` (main output to chat)

**Retrieval Workflow:**
- **Replaced Cohere rerank-v3.5 with Voyage AI rerank-2.5** — new "Rerank Voyage AI" HTTP Request node (`top_k: 15`). Voyage response format matches Cohere, no downstream code changes needed
- **Fixed If node type validation error** — changed from `object notEmpty` (strict) to `string notEmpty` (loose) to handle empty string from sub-workflow
- **Fixed expression prefix on Query Tabular Rows** — `{{ $fromAI('sql_query') }}` → `={{ $fromAI('sql_query') }}` on both SQL tool nodes
- **Upgraded Agentic RAG 1 agent node** — v1.9 → v3.1 with explicit `maxIterations: 25`
- **Added toolDescription to Dynamic Hybrid Search** — concise action-oriented description for agent tool selection
- **Removed orphaned nodes** — "Create Zep User" (disconnected), 4 unused sticky notes

**Ingestion Workflow:**
- **Fixed Insert into Vector Store expression** — `"query": "{{ $json.insertQuery }}"` → `"={{ $json.insertQuery }}"` (missing `=` prefix treated value as literal string)

**Both Workflows:**
- **Bumped 49 node typeVersions** — If (2.2→2.3), Switch (3.2→3.4), HTTP Request (4.2→4.4), Code (2→2.1), Extract From File (1→1.1), chainLlm (1.6→1.9), and others

### v0.2.0 - 2026-02-25

**Pipeline Optimizations:**
- **Added `contextual_summary` metadata field** — the Haiku-generated contextual summary is now stored as a separate `contextual_summary` key in the chunk's metadata JSONB, in addition to being prepended to the content. This gives the retrieval agent clean access to the summary without parsing it from the content, and enables future use in reranking and FTS weighting. Three nodes modified: `Chunk with contextual embedding` (extracts summary), `Setup Chunk for Embedding1` (carries it through batching), `Setup Chunk for Batch Insertion` (merges into metadata)
- **Reduced `batchInterval` from 1000ms to 500ms** — prompt cache hit rate remains high at 94.4% (vs 98.6% at 1000ms) while reducing total ingestion time
- **Removed Wait 5s node** — no longer needed since `batchInterval` on the HTTP Request node handles inter-request pacing
- **Fixed If3 type validation error** — changed condition from `$json` (object notEmpty, strict) to `$json.metadata_name` (string notEmpty, loose). The Supabase metadata query result was failing strict object type validation when receiving empty data
- **Set workflow execution order to v1** — explicit execution order setting

### v0.1.9 - 2026-02-24

**LlamaParse Integration (re-enabled):**
- **Fixed LlamaParse upload endpoint** — changed from `/api/v1/files/` (file management API) to `/api/parsing/upload` (parsing API) which uploads and starts the parsing job in one step
- **Fixed upload form data** — changed parameter type from `formData` to `formBinaryData` and field name to `file` so the actual binary file is sent instead of a string literal
- **Re-enabled full LlamaParse polling chain** — enabled all 7 nodes: Upload, Wait1, Set counter, Is Job Ready?, Wait, Get Processing Status, Counter, Get parsed document
- **Word documents (.doc/.docx) now supported** — routed via LlamaParse through the "Other doc types" branch in Switch1

**Model & Architecture Changes:**
- **Switched contextual embeddings to Anthropic Claude Haiku 4.5** — replaced OpenAI gpt-5-nano with Claude Haiku 4.5 for per-chunk contextual descriptions
- **Implemented Anthropic prompt caching** — replaced `Basic LLM Chain` + `Anthropic Chat Model` LangChain nodes with a direct `Contextual Embedding API` HTTP Request node calling the Anthropic Messages API. The full document is passed as a system message with `cache_control: { type: 'ephemeral' }`, so it's cached across all chunk iterations within a batch. Uses raw body mode (`contentType: raw`) to prevent n8n's JSON parse/re-serialize cycle from breaking cache key matching. Sequential processing (`batchSize: 1`) ensures each request can read the previous request's cache. Estimated ~89% input token cost reduction on documents exceeding 4,096 tokens
- **Removed duplicate contextual embedding LLM chain** — `Basic LLM Chain1` was running the same prompt as `Basic LLM Chain`, doubling API calls and token usage per chunk. Removed the duplicate
- **Stricter contextual embedding prompt** — enforced `Context:` prefix format with explicit rules and example output

**Bug Fixes:**
- **Fixed prompt caching not activating** — n8n's `specifyBody: "json"` mode parses the JSON string back into an object then re-serializes, potentially breaking Anthropic's byte-level cache key matching. Switched to `contentType: "raw"` with `rawContentType: "application/json"` so the `JSON.stringify()` output is sent as-is without re-serialization
- **Fixed HTTP Request parallel execution breaking cache** — n8n's HTTP Request node processes items in parallel by default (batch of 50). Added `batchSize: 1` to force sequential processing so each request can read the cache created by the previous one
- **Fixed Recycling Bin deletion failing** — the `If` node in the deletion flow had strict type validation on `$json.id` (number exists) which failed when Supabase returned empty/string data. Changed to loose type validation

### v0.1.8c - 2026-02-19

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
- Added **RAG Retrieval Sub-Workflow** — agentic retrieval with dynamic hybrid search
- Added **Knowledge Graph Workflow** — knowledge graph sub-workflow
- Added **Multimodal RAG Ingestion Sub-workflow** — OCR and image processing
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
