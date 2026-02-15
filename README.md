# Modular RAG

A production-grade n8n workflow for document ingestion into a RAG (Retrieval-Augmented Generation) pipeline. Monitors Google Drive (and accepts webhooks), extracts content from multiple file formats, chunks intelligently, generates embeddings, and stores everything in a PostgreSQL vector database.

## Architecture

```
Triggers (Google Drive / Webhook)
  -> File Type Detection (Switch1)
  -> Format-Specific Extraction
  -> Record Manager Dedup (SHA256 hash check)
  -> LLM Enrichment (headline, summary, custom metadata)
  -> Hierarchical Markdown-Aware Chunking
  -> Embedding Generation (OpenAI text-embedding-3-small)
  -> Vector Store Insertion (PostgreSQL pgvector)
  -> Status Update & File Archival
```

## Supported File Types

| Category | Formats |
|----------|---------|
| Text | Plain Text, Markdown |
| Documents | Google Docs, PDF, HTML, Word (.doc/.docx), PowerPoint, RTF, EPUB, and more |
| Spreadsheets | Excel (.xlsx), CSV, Google Sheets |
| Media | Images (JPEG, PNG, GIF, etc.), Audio (MP3, WAV), Video (MP4, WebM) |

## Database Tables

| Table | Purpose |
|-------|---------|
| `record_manager_v2` | Document tracking, deduplication, processing status |
| `documents_v2` | Vector embeddings (content, metadata, embedding via pgvector) |
| `tabular_document_rows` | Row-level data from Excel/CSV/Sheets |
| `metadata_fields` | Custom metadata field definitions for LLM enrichment |

## Key Features

- **Smart Chunking** - Hierarchical markdown-aware splitting (min 400 / target 600 / max 800 chars) with page number tracking
- **Deduplication** - SHA256 content hashing via `record_manager_v2`; skips unchanged docs, re-indexes changed ones
- **LLM Enrichment** - GPT-4 mini extracts headline, summary, and custom metadata per document
- **Batch Embedding** - Chunks batched (200/batch) with OpenAI `text-embedding-3-small`, inserted via PostgreSQL `UNNEST`
- **Tabular Data** - Excel/CSV rows stored individually as JSON in `tabular_document_rows`, optionally embedded

## Optional Features (Toggle via Set Data node)

| Flag | Purpose |
|------|---------|
| `contextual_embedding_enabled` | GPT-4 mini generates contextual snippets per chunk before embedding |
| `lightrag_enabled` | Routes to LightRAG sub-workflow for knowledge graph construction |
| `multimodal_rag_enabled` | Routes to multimodal sub-workflow for image/visual content |
| `ocr_enabled` | Mistral AI OCR for scanned documents and images |
| `send_tabular_data_to_vector_store` | Also embed tabular rows into the vector store |

## External Services

- **Google Drive** - File trigger (new/updated files) + file operations
- **OpenAI** - Embeddings (`text-embedding-3-small`) + LLM enrichment (GPT-4 mini)
- **Mistral AI** - OCR (`mistral-ocr-latest`)
- **LlamaParse** - Advanced document parsing (disabled by default)
- **Supabase / PostgreSQL** - Vector store + record management
- **Firecrawl** - Web scraping (disabled by default)

## Triggers

- **Google Drive - New Files** - Monitors "Rag Files" folder, polls every 1 min
- **Google Drive - Updated Files** - Same folder, polls every 1 min
- **Webhook** - HTTP POST endpoint for programmatic ingestion

---

## Changelog

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
