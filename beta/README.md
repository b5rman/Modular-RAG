# Modular RAG — Beta Workflows

Beta/experimental workflows for the [Modular RAG](https://github.com/b5rman/Modular-RAG) system. These workflows test new features before they are merged into production.

**Production base version:** v0.2.0 (ingestion) / v1.0.8c (retrieval)

## Workflows

| File | n8n Workflow ID | Description |
|------|----------------|-------------|
| `RAG Retrieval Sub-Workflow (Beta).json` | `EQTUC67VhmFzuiAB` | Retrieval agent with Citation Verification System + extended thinking |
| `RAG INGESTION (Beta - Batch Contextual).json` | — | Ingestion pipeline with batch contextual embedding route |
| `Contextual Embedding Batch (Beta).json` | — | OpenAI Batch API sub-workflow for async contextual embeddings |

## RAG Retrieval Sub-Workflow (Beta)

The primary beta workflow. Replaces the production retrieval agent with significant upgrades.

### Changes from Production

| Feature | Production (v1.0.8c) | Beta |
|---------|---------------------|------|
| **Model** | GPT-5.2 | Claude Sonnet 4.6 |
| **Extended thinking** | No | Yes (budget: 10,000 tokens) |
| **Citation format** | Vague "list 1-5 references" | Structured JSON with validation |
| **Response mode** | Streaming | lastNode (waits for full response) |
| **Post-processing** | None | Format & Verify Citations code node |
| **Tabular SQL** | Disabled | Enabled |

### Citation Verification System (Phase 1)

The agent outputs a structured format with two sections separated by a `---SOURCES_JSON---` delimiter:

**Section 1 — Answer:** Full markdown response (no inline references section).

**Section 2 — Sources JSON:** Array of source objects, each requiring:
- `doc_name` — Document name from chunk metadata
- `doc_id` — The `record_manager_id` from chunk metadata
- `pages` — Array of page numbers referenced
- `chunk_indices` — Array of `chunk_index` values from chunks used
- `relevance` — Short description of what the source contributed

A downstream **Format & Verify Citations** code node:
1. Splits the output on the delimiter
2. Parses and validates each source object (required fields, correct types)
3. Strips any References section the agent may have included in the answer
4. Builds a clean `## References` section from validated sources
5. Tracks `citationWarnings` for debugging (e.g., missing fields, parse failures)
6. Falls back gracefully if no delimiter is found

### Architecture

```
Chat Trigger (lastNode mode)
    │
    ▼
Agentic RAG 1 Beta (Sonnet 4.6 + extended thinking)
  ├── ai_languageModel: Anthropic Chat Model (Sonnet 4.6)
  ├── ai_memory: Supabase Short-Term Memory
  ├── ai_tool: Dynamic Hybrid Search
  ├── ai_tool: Fetch Document Hierarchy
  ├── ai_tool: Context Expansion
  └── ai_tool: Query Tabular Rows
    │
    ▼ (main output)
Format & Verify Citations (Code node)
    │
    ▼ (final output returned to chat)
```

### Test Results

| Execution | Result | Details |
|-----------|--------|---------|
| #3495 | Fix | `maxTokensToSample` must exceed `thinkingBudget` — set to 16,000 |
| #3496 | Pass | No-results path: empty sources array, fallback References section |
| #3508 | Pass | Single-document: 1 source, 19 chunk_indices, zero warnings, 44s |
| Cross-ref | Pass | 3 documents, 3 source objects, 35 chunk_indices, zero warnings |

## Contextual Embedding Batch API

> **Note:** Production (v0.2.0) now uses synchronous Anthropic Haiku 4.5 with prompt caching via direct HTTP Request, which is faster and cheaper than the batch approach below. This batch workflow remains as an alternative for very large ingestion jobs where async processing is acceptable.

### How It Works

**Phase 1 — Submit** (triggered by ingestion pipeline):
1. Receives document + chunks + metadata from RAG Ingestion
2. Converts chunks to `/v1/chat/completions` JSONL batch (gpt-4o-mini)
3. Uploads JSONL to OpenAI Files API
4. Creates batch job via OpenAI Batch API
5. Stores batch ID + original chunks/metadata in `openai_batches` table

**Phase 2 — Poll, Embed & Insert** (cron every 7 mins):
1. Queries `openai_batches` for uncompleted batches
2. Checks batch status via OpenAI API
3. If completed: downloads `.jsonl` result, pairs with original chunks
4. Creates embeddings via `text-embedding-3-small`
5. Inserts into `documents_v2` vector store via PostgreSQL `UNNEST`

### Database Table: `openai_batches`

| Column | Type | Description |
|--------|------|-------------|
| `id` | text (PK) | OpenAI batch ID |
| `batch_status` | text | `validating`, `in_progress`, `completed`, `failed`, `expired`, `cancelled` |
| `output_file_id` | text | OpenAI file ID for completed results |
| `result` | text[] | Array of contextual descriptions |
| `input_data` | text | JSON string with original chunks and metadata |
| `created_at` | timestamptz | Row creation time |
| `updated_at` | timestamptz | Last status update |

## Prompts

The `prompts/` folder contains the current system prompts and tool descriptions used by the beta retrieval agent:

| File | Description |
|------|-------------|
| `agentic-rag-1-system-prompt.md` | Full agent system prompt with citation output format |
| `contextual-embedding-prompt.md` | Contextual embedding generation prompt |
| `dynamic-hybrid-search-tool-description.md` | Hybrid search tool description |
| `query-tabular-rows-tool-description.md` | SQL tabular query tool description |

## Backups

The `backups/` folder contains pre-edit snapshots of workflow files for rollback purposes.

---

## Changelog

### 2026-02-25 — Citation Verification System Phase 1

**New: Citation Verification pipeline**
- Added `Format & Verify Citations` code node — validates structured citations, builds References section
- Agent system prompt rewritten with `---SOURCES_JSON---` delimiter output format
- Each source requires: `doc_name`, `doc_id`, `pages[]`, `chunk_indices[]`, `relevance`

**Changed: Model upgrade**
- Switched from GPT-5.2 to Claude Sonnet 4.6 with extended thinking (budget: 10,000 tokens)
- Set `maxTokensToSample: 16000` to support extended thinking

**Changed: Response mode**
- Switched chatTrigger from streaming to `lastNode` — required for citation post-processing
- Users wait for full response (~30-45s) instead of seeing streaming tokens

**Removed: Orphaned nodes**
- Removed disconnected nodes: Agentic RAG 1 (old), Create Zep User, Dynamic Hybrid Search (old), Query Tabular Rows (old)

### 2026-02-21 — Beta-0.1.9: Batch Contextual Embeddings

**New: Contextual Embedding Batch API sub-workflow**
- OpenAI Batch API for async contextual embeddings (50% cheaper, separate rate limits)
- Phase 1 (submit) + Phase 2 (poll/embed/insert) architecture
- New `openai_batches` Supabase table for job tracking

**Modified: RAG Ingestion pipeline**
- Added Execute Workflow node to route chunks to batch sub-workflow

**Modified: Retrieval Sub-Workflow**
- Enabled `Query Tabular Rows` tool for structured data retrieval via SQL
