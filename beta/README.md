# Modular RAG — Beta

Beta branch for testing new features before merging to production (`master`).

**Current version:** Beta-0.1.9
**Base version:** v1.0.8c (production)
**Branch:** `beta`

## What's New in Beta-0.1.9

**Contextual Embedding Batch API** — Replaces the throttled per-chunk contextual embedding pipeline with OpenAI's Batch API. Instead of calling gpt-4o-mini once per chunk (with 5s waits and batch size 4), all chunks are submitted in a single batch job, then embedded and inserted into the vector store when complete.

Key benefits:
- **50% cheaper** than real-time API calls
- **Separate rate limits** — no more throttling
- **Submit hundreds of chunks at once**
- **Trade-off:** Async processing (up to 24h, typically minutes)

**Tabular SQL Queries** — Enabled `Query Tabular Rows` as an active retrieval tool. The agent can now query the `tabular_document_rows` table directly via SQL to answer questions about structured data (Excel/CSV/Sheets). Previously disabled in production.

## Workflows

| File | n8n Workflow | Description |
|------|-------------|-------------|
| `RAG INGESTION (Beta - Batch Contextual).json` | RAG INGESTION v0.1.8c (Beta) | Ingestion pipeline modified to route chunks to the Batch API sub-workflow |
| `Contextual Embedding Batch (Beta).json` | Contextual Embedding Batch API (Beta) | **New** — OpenAI Batch API for contextual embeddings (submit + poll + embed + insert) |
| `RAG Retrieval Sub-Workflow (Beta).json` | RAG Retrieval Sub-Workflow v1.0.8c BETA | Retrieval agent with tabular SQL queries enabled |

## Contextual Embedding Batch API

### How It Works

**Phase 1 — Submit** (triggered by ingestion pipeline):
1. Receives document + chunks + metadata from RAG Ingestion
2. Converts chunks to `/v1/chat/completions` JSONL batch (gpt-4o-mini, max 256 tokens per context)
3. Uploads JSONL to OpenAI Files API
4. Creates batch job via OpenAI Batch API
5. Stores batch ID + original chunks/metadata in `openai_batches` table

**Phase 2 — Poll, Embed & Insert** (cron every 7 mins):
1. Queries `openai_batches` for uncompleted batches
2. Checks batch status via OpenAI API
3. If completed: downloads `.jsonl` result, decodes contextual descriptions
4. Pairs descriptions with original chunks → enriched content
5. Splits into batches of 20, creates embeddings via `text-embedding-3-small`
6. Inserts into `documents_v2` vector store via PostgreSQL `UNNEST`
7. Updates `openai_batches` row with status, output file ID, and results

### Triggers
- **Sub-Workflow Trigger** — Called from RAG Ingestion pipeline (primary)
- **Manual Trigger** — For standalone testing with pinned data
- **Cron (7 min interval)** — Polls for completed batches

## Database Changes

### New Table: `openai_batches`

| Column | Type | Description |
|--------|------|-------------|
| `id` | text (PK) | OpenAI batch ID (e.g., `batch_699a07ef...`) |
| `batch_status` | text | Batch status (`validating`, `in_progress`, `completed`, `failed`, `expired`, `cancelled`) |
| `output_file_id` | text | OpenAI file ID for completed batch results |
| `result` | text[] | Array of contextual descriptions (populated on completion) |
| `input_data` | text | JSON string containing original chunks, chunk_metadata, and record_manager_id |
| `created_at` | timestamptz | Row creation time (default: `now()`) |
| `updated_at` | timestamptz | Last status update time |

---

## Changelog

### Beta-0.1.9 — 2026-02-21

**New: Contextual Embedding Batch API sub-workflow**
- Added `Contextual Embedding Batch API (Beta)` — new sub-workflow using OpenAI Batch API for contextual embeddings
- Phase 1 (submit): converts chunks to JSONL, uploads to OpenAI, creates batch, stores in `openai_batches` table
- Phase 2 (poll): cron checks for completed batches, downloads results, creates embeddings (`text-embedding-3-small`), inserts into `documents_v2` via PostgreSQL `UNNEST`
- Embedding batches of 20 chunks with retry logic (5 retries, 2.5s wait)

**Modified: RAG Ingestion pipeline**
- Added Execute Workflow node to route chunks to the Contextual Embedding Batch API sub-workflow
- Ingestion now supports async contextual embeddings via batch processing

**Modified: Retrieval Sub-Workflow**
- Enabled `Query Tabular Rows` tool — agent can now query `tabular_document_rows` via SQL for structured data retrieval (Excel/CSV/Sheets)
- Production has this disabled (`Query Tabular Rows1` and `Query Tabular Rows2` both disabled)

**Database:**
- New `openai_batches` table in Supabase for batch job tracking
