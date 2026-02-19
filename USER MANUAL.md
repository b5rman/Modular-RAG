# Modular RAG — User Manual

## Table of Contents

1. [Getting Started](#getting-started)
2. [Uploading Documents](#uploading-documents)
3. [Supported File Types](#supported-file-types)
4. [Asking Questions (Chat)](#asking-questions-chat)
5. [Updating Documents](#updating-documents)
6. [Deleting Documents](#deleting-documents)
7. [How Ingestion Works](#how-ingestion-works)
8. [Troubleshooting](#troubleshooting)

---

## Getting Started

This system has two main workflows that need to be running in n8n:

| Workflow | What it does | Must be active? |
|----------|-------------|-----------------|
| **RAG INGESTION** | Processes files you upload and stores them in the knowledge base | Yes |
| **Retrieval Sub-Workflow** | Answers your questions using the knowledge base | Yes |

### Activating the Workflows

For each workflow:

1. Open the workflow in n8n
2. Toggle the **Active** switch in the top-right corner to **ON**
3. The toggle should turn green — this means the workflow is live

**RAG INGESTION** needs to be active so it can monitor your Google Drive folder for new and updated files (polls every minute).

**Retrieval Sub-Workflow** needs to be active so the chat interface is available. You can test it inside the n8n editor without activating (click the Chat bubble in the bottom-right), but for regular use it must be active.

---

## Uploading Documents

### Step 1 — Upload to Google Drive

Upload your files directly into the **"Rag Files"** folder in Google Drive.

> **Important:**
> - Upload files **directly** into the folder — do not move files from another folder. Google Drive triggers do not fire when you move files between folders.
> - The system processes **one file at a time**. If you upload multiple files, they will be queued and processed sequentially.
> - The folder is polled **every 1 minute**, so there may be a short delay before processing starts.

### Step 2 — Wait for Processing

Once a file is detected:
1. The system extracts the content based on file type (including OCR for scanned documents)
2. The content is split into smart chunks (preserving document structure)
3. An AI enriches each document with a headline, summary, and metadata
4. Chunks are embedded using OpenAI and stored in the vector database
6. The original file is **archived** (moved to an archive folder in Google Drive)

You don't need to do anything — just upload and wait.

### Step 3 — Verify (Optional)

You can verify your document was processed by asking the chat a question about its content.

---

## Supported File Types

| Category | Formats |
|----------|---------|
| **Text** | Plain Text (.txt), Markdown (.md) |
| **Documents** | PDF, Google Docs, HTML |
| **Spreadsheets** | Excel (.xlsx), CSV, Google Sheets |

> **Not currently supported:** Word (.doc/.docx), PowerPoint, RTF, EPUB, images, audio, and video files require LlamaParse which is not enabled. These files will be moved to the Error folder if uploaded.

### Notes on Spreadsheets (Excel, CSV, Google Sheets)

Tabular data (rows and columns) is handled differently from regular documents:
- Each row is stored individually as a JSON record
- Rows are also embedded into the vector store so the AI can search them
- The AI can find specific data points, sums, and values from your spreadsheets

---

## Asking Questions (Chat)

### Opening the Chat

1. Open the **Retrieval Sub-Workflow** in n8n
2. Click the **Chat** button (speech bubble icon) in the bottom-right corner
3. Type your question and press Enter

### How the AI Answers

The AI agent uses a multi-step retrieval process:

1. **Hybrid Search** — Searches the knowledge base using a combination of semantic (meaning-based) and lexical (keyword-based) search. The agent dynamically adjusts search weights based on query type
2. **Reranking** — Results are reranked by relevance using Cohere for higher accuracy
3. **Document Hierarchy** — Loads the structure of the source document
4. **Context Expansion** — Retrieves surrounding content for more complete answers

The AI will provide answers based on retrieved documents from the knowledge base.

### Tips for Better Results

- **Be specific** — "What are the Q3 revenue figures from the financial report?" works better than "Tell me about revenue"
- **Reference document names** if you know them — "According to the project plan document, what are the milestones?"
- **For spreadsheet data**, ask direct questions — "What is the total sales amount in the Excel file?"
- If the AI says it doesn't have enough information, the document may not have been ingested yet

---

## Updating Documents

To update a document that's already in the knowledge base:

1. **Upload the updated version** to the **"Rag Files"** Google Drive folder with the **same filename**
2. The system will automatically:
   - Detect that the document has changed (via content hashing)
   - Delete the old vectors from the database
   - Re-process and store the updated content

> **Important:** Upload the file — do not edit it in place on Google Drive. The trigger watches for file creation and update events in the folder.

---

## Deleting Documents

To remove a document from the knowledge base:

1. Move the file into the **"Recycling Bin"** sub-folder inside your Google Drive RAG folder
2. The system runs a scheduled check **every 2 minutes** that:
   - Finds all files in the Recycling Bin
   - Deletes their vectors from the database
   - Deletes the record from the record manager
   - Removes the file from Google Drive

> **Note:** Deletion is not instant — it runs on a 2-minute schedule. After the next cycle, the document will no longer appear in search results.

---

## How Ingestion Works

Here's what happens behind the scenes when you upload a file:

```
Upload file to "Rag Files" folder
        |
        v
  File detected (polled every 1 min)
        |
        v
  Check record manager — is this file new or changed?
        |
   +---------+-----------+
   |         |           |
  New     Changed    Unchanged
   |         |           |
   v         v           v
 Extract   Delete old   Skip
 content   vectors,     (move to
   |       re-extract   next file)
   v         |
 Smart chunking (preserves headings, pages)
   |
   v
 AI enrichment (headline, summary, metadata)
   |
   v
 Generate embeddings (OpenAI)
   |
   v
 Store in vector database
   |
   v
 Archive original file
```

---

## Troubleshooting

### "I uploaded a file but the AI can't find it"

- **Wait 1-2 minutes** — the folder is polled every minute, and processing takes time
- Check that the **RAG INGESTION** workflow is **Active** (toggled on) in n8n
- Make sure you uploaded directly to the **"Rag Files"** folder (not a subfolder or moved from elsewhere)
- Check the n8n execution log for errors (click **Executions** in the left sidebar)

### "The AI gives wrong or incomplete answers"

- Try rephrasing your question with more specific terms
- The AI only answers based on what's in the knowledge base — it will not make things up
- If a document was recently updated, make sure the new version has been processed

### "Spreadsheet data isn't found"

- Excel/CSV/Google Sheets data needs to be re-ingested after enabling tabular vector storage
- If the data was uploaded before this feature was enabled, delete the file (move to Recycling Bin), wait for deletion, then re-upload

### "Google Sheets shows a Forbidden error"

- The Google OAuth token may need refreshing — open the credential in n8n and re-authenticate

### Processing seems stuck

- Check the n8n execution log for the specific error
- The system has retry logic (up to 5 retries) for transient failures
- If a file causes a persistent error, it will be moved to an **Error folder** in Google Drive
