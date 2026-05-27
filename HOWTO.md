---
name: AgentMemory Setup Guide — Hybrid Search (BM25 + Vector)
version: 2.0.0
date: 2026-05-27
author: Teddy Oswari (fork of toswari/agentmemory)
description: Complete guide for configuring agentmemory with LM Studio embeddings, BM25+Vector hybrid search, and integration with Hermes Agent + OpenCode
---

# AgentMemory Setup Guide — Hybrid Search (BM25 + Vector)

## Overview

This fork of [toswari/agentmemory](https://github.com/toswari/agentmemory) enables **zero-cost, offline embeddings** via [LM Studio](https://lmstudio.ai/) and **hybrid search** combining BM25 keyword matching with vector semantic similarity for superior memory retrieval.

### Why Use This Setup?

- **Zero API costs** — all embedding generation happens locally
- **Privacy** — no text data leaves your machine
- **Hybrid search accuracy** — combines BM25 (keyword) + Vector (semantic) scoring
- **Offline capability** — works without internet once models are downloaded
- **Full agent integration** — MCP server connects to Hermes Agent and OpenCode

---

## Prerequisites

### 1. LM Studio Installation

Download and install [LM Studio](https://lmstudio.ai/) for macOS, Windows, or Linux.

### 2. Load an Embedding Model in LM Studio

Open LM Studio and search for **nomic-embed-text** models:

| Model | Dimensions | Use Case |
|-------|------------|----------|
| `text-embedding-nomic-embed-text-v1.5` | 768 | Default recommendation |
| `text-embedding-nomic-embed-text-v1.4` | 768 | Older version |

**Steps:**
1. Open LM Studio → Browse tab
2. Search for "nomic-embed-text"  
3. Select a model and click **Download**
4. Once downloaded, select it in the dropdown at top-left
5. Start serving (server icon or `Server` button) on port **1234**

### 3. Verify LM Studio is Running

```bash
# Check models are available
curl http://localhost:1234/v1/models

# Test embedding endpoint directly
curl -X POST http://localhost:1234/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{
    "model": "text-embedding-nomic-embed-text-v1.5",
    "input": ["test"],
    "encoding_format": "float"
  }' | python3 -m json.tool
```

Expected output should show a **768-dimensional vector** array.

---

## Step 1: Environment Configuration

### Add to Your Shell Config (`~/.zshrc` or `~/.bashrc`)

```bash
# >>> agentmemory — Hybrid Search & MCP Configuration <<<
# Enable all MCP tools (default shows only core 8)
export AGENTMEMORY_TOOLS=all

# Hybrid search weights: BM25=0.4, VECTOR=0.6 (semantic > keyword by default)
export BM25_WEIGHT=0.4
export VECTOR_WEIGHT=0.6

# Search precision tuning
export SEARCH_TOP_K=10
export SEARCH_MIN_SCORE=0.3

# Embedding provider for agentmemory vector search
export OPENAI_BASE_URL=http://localhost:1234/v1
export OPENAI_API_KEY=not-needed
export OPENAI_EMBEDDING_MODEL=text-embedding-nomic-embed-text-v1.5
export OPENAI_EMBEDDING_DIMENSIONS=768
export EMBEDDING_PROVIDER=openai

# <<< agentmemory Configuration >>>
```

> **Note:** `EMBEDDING_PROVIDER=openai` forces the OpenAI provider selection, which then routes to your local LM Studio via `OPENAI_BASE_URL`.

### Source Your Shell Config

```bash
source ~/.zshrc   # or source ~/.bashrc
```

---

## Step 2: Start AgentMemory MCP Server

A control utility has been created at `~/.local/bin/agentmemory-ctl` to manage the MCP server.

### Basic Usage

```bash
# Start the MCP memory server
agentmemory-ctl start

# Check if it's running
agentmemory-ctl status

# Stop the server
agentmemory-ctl stop
```

### Status Output Example

```
[agentmemory] Running (PID: 12345)
  Port 3111: Bound and listening
  Health check: HTTP 200
  12345    01:23:45 agentmemory
```

### Where to Find Logs

Server output is captured to `~/.local/log/agentmemory.log`:

```bash
tail -f ~/.local/log/agentmemory.log
```

---

## Step 3: Configure Your Agent (Hermes or OpenCode)

### Option A: Hermes Agent Configuration

Add this to `~/.hermes/config.yaml`:

```yaml
mcp_servers:
  agentmemory:
    url: http://localhost:3111/mcp
    enabled: true
    tools_filter: all   # Loads all 53 MCP tools (default shows only core 8)
  gitnexus:             # Optional — code intelligence MCP server
    url: http://localhost:4747/mcp
    enabled: true
    tools_filter: core
```

After updating config, restart Hermes to reload MCP servers.

### Option B: OpenCode Configuration

OpenCode connects via the SDK dynamically and inherits environment variables automatically. Ensure these are active in your shell before launching opencode:

```bash
# Environment variables from ~/.zshrc handle this automatically
AGENTMEMORY_TOOLS=all \
BM25_WEIGHT=0.4 \
VECTOR_WEIGHT=0.6 \
OPENAI_BASE_URL=http://localhost:1234/v1 \
EMBEDDING_PROVIDER=openai \
opencode <your-command>
```

Or simply run opencode normally — env vars are inherited from your shell profile:

```bash
# Just run opencode (env vars already in ~/.zshrc)
opencode <command>
```

---

## Step 4: Verify Hybrid Search Configuration

### Verify LM Studio Embedding Endpoint

```bash
curl -X POST http://localhost:1234/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"model":"text-embedding-nomic-embed-text-v1.5","input":["test"],"encoding_format":"float"}'
```

Expected output should show a **768-dimensional vector** array.

### Verify AgentMemory MCP Server Health

```bash
curl http://localhost:3111/health
```

Should return HTTP 200 with status information.

### Verify Hybrid Search Weights Are Active

Check the agentmemory logs for startup confirmation:

```bash
tail ~/.local/log/agentmemory.log | grep -E "BM25|VECTOR|Embedding"
```

Expected output includes:
- `Embedding provider: openai` (or your configured provider)
- `BM25 weight: 0.4`
- `Vector weight: 0.6`
- `AGENTMEMORY_TOOLS=all loaded, enabling all tools`

### Test Hybrid Search Directly

```bash
curl -X POST http://localhost:3111/agentmemory/search \
  -H "Content-Type: application/json" \
  -d '{"query":"database migration patterns","top_k":5}'
```

Response should include both `vectorScore` and `bm25Score` for each result.

---

## How Hybrid Search Works

The `memory_smart_search` tool combines two retrieval methods:

### 1. Vector (Semantic) Search
Uses embeddings to find conceptually similar content regardless of exact keywords.

### 2. BM25 (Keyword) Search  
Uses TF-IDF-like scoring for exact keyword matches — great for finding specific terms like function names, error codes, or API endpoints.

### Weighted Scoring Formula

```
FinalScore = VECTOR_WEIGHT * semantic_score + BM25_WEIGHT * keyword_score
```

With defaults of **0.6 (vector) / 0.4 (BM25)**, semantic search is given more weight, but both contribute to results.

### Tuning Guidelines

| Use Case | BM25_WEIGHT | VECTOR_WEIGHT | Why |
|----------|-------------|---------------|-----|
| Technical code queries | 0.5 | 0.5 | Keywords matter for function names, APIs |
| Conceptual/explanatory queries | 0.3 | 0.7 | Semantics better for abstract questions |
| Debugging specific errors | 0.6 | 0.4 | Error messages are keyword-heavy |

---

## Architecture Diagram

```
┌─────────────────┐     ┌──────────────────────┐     ┌─────────────────┐
│   Agent/CLI     │     │  LM Studio           │     │   SQLite        │
│                 │     │  (Port 1234)         │     │  Vector Index   │
│  Hermes/Open-   │────▶│                      │     │  (Local DB)     │
│  Code           │◀────│  Embedding Model:    │◀────│                 │
└────────┬────────┘     │  nomic-embed-text   │     └─────────────────┘
         │              └──────────────────────┘           ^
         │                                                 |
         ▼                                                 |
┌─────────────────────────────────────────────────────────┐
│              agentmemory MCP Server (Port 3111)          │
│                                                         │
│  Tools Available:                                       │
│  • memory_smart_search   ← Hybrid BM25 + Vector search  │
│  • memory_recall         ← Recall past sessions        │
│  • memory_remember       ← Store new memories           │
│  • memory_forget         ← Remove specific memories     │
│  • memory_handoff        ← Share context across agents  │
│  • memory_recap          ← Session summary              │
│  • ... (53 tools total with AGENTMEMORY_TOOLS=all)      │
└────────┬────────────────────────────────────────────────┘
         │
    Query/Recall
         ▼
                                                      Retrieve
```

---

## Troubleshooting

### LM Studio Not Responding

```bash
# Check if LM Studio is running
curl -v http://localhost:1234/v1/models

# If fails, start LM Studio manually and verify port
lsof -i :1234 | grep LISTEN
```

### Wrong Dimension Error

If you see errors like `Embedding dimension mismatch in openai.embed`:

- Verify your model matches the configured dimension (768 for nomic)
- Check `OPENAI_EMBEDDING_DIMENSIONS` is set correctly
- Try a different LM Studio embedding model if needed

### AgentMemory Can't Connect to MCP Server

```bash
# Check server status
agentmemory-ctl status

# Check if port 3111 is bound
lsof -i :3111 | grep LISTEN

# Restart if needed
agentmemory-ctl stop && agentmemory-ctl start
```

### Model Not Found in LM Studio

Verify the exact model name in LM Studio's list:

```bash
curl http://localhost:1234/v1/models | python3 -c "import json, sys; [print(m['id']) for m in json.load(sys.stdin)['data'] if 'embed' in m['id'].lower()]"
```

Update `OPENAI_EMBEDDING_MODEL` to match the exact model ID.

### Hybrid Search Not Working (Only One Score)

If results only show either vectorScore OR bm25Score but not both:

1. Verify both weights are non-zero: `echo $BM25_WEIGHT && echo $VECTOR_WEIGHT`
2. Check agentmemory logs for errors during search initialization
3. Ensure AGENTMEMORY_TOOLS=all is set (some tool subsets disable hybrid search)

---

## Quick Reference

### Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `AGENTMEMORY_TOOLS` | `core` | MCP tools to load (`all`, `core`, or comma-separated list) |
| `BM25_WEIGHT` | `0.4` | Weight for BM25 keyword scoring (0-1) |
| `VECTOR_WEIGHT` | `0.6` | Weight for vector semantic scoring (0-1) |
| `SEARCH_TOP_K` | `10` | Max results to return |
| `SEARCH_MIN_SCORE` | `0.3` | Minimum combined score threshold |
| `OPENAI_BASE_URL` | — | LM Studio endpoint URL |
| `OPENAI_API_KEY` | `not-needed` | Bypass auth for local LM Studio |
| `OPENAI_EMBEDDING_MODEL` | — | Model name in LM Studio |
| `OPENAI_EMBEDDING_DIMENSIONS` | `768` | Vector dimension count |
| `EMBEDDING_PROVIDER` | Auto-detected | Force provider selection |

### Commands Reference

```bash
# Shell config reload
source ~/.zshrc

# Start agentmemory MCP server
agentmemory-ctl start

# Check status
agentmemory-ctl status

# Stop agentmemory MCP server
agentmemory-ctl stop

# View logs
tail -f ~/.local/log/agentmemory.log

# Test LM Studio embedding directly
curl -X POST http://localhost:1234/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"model":"text-embedding-nomic-embed-text-v1.5","input":["hello"],"encoding_format":"float"}'

# Test hybrid search via MCP server
curl -X POST http://localhost:3111/agentmemory/search \
  -H "Content-Type: application/json" \
  -d '{"query":"your query here","top_k":5}'

# Verify Hermes MCP connection (in new session)
hermes mcp test agentmemory
```

---

## Files Modified During Setup

| File | Changes |
|------|---------|
| `~/projects/agentmemory/.env.example` | Added Section 11 — Hybrid Search Configuration |
| `~/.hermes/config.yaml` | Added `mcp_servers.agentmemory` section with tools_filter |
| `~/.zshrc` (or `~/.bashrc`) | Added environment variables for hybrid search + embeddings |

---

## Architecture Notes

### Vector Storage

agentmemory uses an **SQLite-backed vector index** (`src/state/vector-index.ts`) for local storage. Vectors are stored as BLOB columns and retrieved via cosine similarity queries. No external database (Chroma, Qdrant, Pinecone) required.

### BM25 Index

BM25 scoring is handled by `src/search/bm25-index.ts` — a lightweight in-memory implementation that computes TF-IDF-style scores without requiring Elasticsearch or other search engines.

### Hybrid Search Flow

```
Query → Split into keywords + semantic representation
     ↓
BM25 index: Calculate keyword relevance scores
Vector index: Retrieve semantic similarity via cosine distance
     ↓
Combine: weighted_sum = BM25_WEIGHT × bm25_score + VECTOR_WEIGHT × vector_score
     ↓
Filter: Apply SEARCH_MIN_SCORE threshold
Sort: Descending by combined score
Return: Top K results (SEARCH_TOP_K) with individual scores exposed
```
