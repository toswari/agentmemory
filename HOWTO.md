---
name: LM Studio Embedding Integration
version: 1.0.0
date: 2026-05-27
author: Teddy Oswari (fork of toswari/agentmemory)
description: Guide for using LM Studio as local embedding provider instead of cloud APIs
---

# LM Studio Embedding Integration for AgentMemory

## Overview

This fork of [toswari/agentmemory](https://github.com/toswari/agentmemory) enables **zero-cost, offline embeddings** by routing through [LM Studio](https://lmstudio.ai/) instead of cloud APIs.

### Why Use This Setup?

- **Zero API costs** — all embedding generation happens locally
- **Privacy** — no text data leaves your machine
- **Offline capability** — works without internet once model is downloaded
- **Full compatibility** — leverages existing OpenAIEmbeddingProvider in agentmemory

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

## Configuration (Option A — Recommended)

Use environment variables to redirect agentmemory's OpenAI embedding calls to LM Studio.

### Add to Your Shell Config

Add these lines to `~/.bashrc` or `~/.zshrc`:

```bash
# >>> agentmemory — LM Studio Embedding Configuration <<<
export OPENAI_BASE_URL=http://localhost:1234/v1
export OPENAI_API_KEY=not-needed
export OPENAI_EMBEDDING_MODEL=text-embedding-nomic-embed-text-v1.5
export OPENAI_EMBEDDING_DIMENSIONS=768
export EMBEDDING_PROVIDER=openai
# <<< agentmemory — LM Studio Embedding Configuration >>>
```

> **Note:** `EMBEDDING_PROVIDER=openai` forces the OpenAI provider selection, which then routes to your local LM Studio via `OPENAI_BASE_URL`.

### Source Your Shell Config

```bash
source ~/.zshrc   # or source ~/.bashrc
```

---

## Using agentmemory-ctl (Recommended)

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

## How It Works (Under the Hood)

### Request Flow

```
agentmemory
    ↓
OpenAIEmbeddingProvider (src/providers/embedding/openai.ts)
    ↓
Uses OPENAI_BASE_URL environment variable
    ↓
http://localhost:1234/v1/embeddings  ← LM Studio endpoint
    ↓
Returns Float32Array vectors to vector-index.ts
```

### Key Files Modified/Referenced

| File | Role |
|------|------|
| `src/providers/embedding/openai.ts` | Uses OPENAI_BASE_URL for requests |
| `src/config.ts` | detectEmbeddingProvider() respects EMBEDDING_PROVIDER override |
| `src/state/vector-index.ts` | Stores/retrieves vectors with cosine similarity |

### Vector Storage

agentmemory uses an **SQLite-backed vector index** (`src/state/vector-index.ts`) for local storage. Vectors are stored as BLOB columns and retrieved via cosine similarity queries. No external database (Chroma, Qdrant, Pinecone) required.

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

---

## Integration with Hermes Agent (Optional)

To use agentmemory as an MCP server within Hermes:

### 1. Start agentmemory Server

```bash
agentmemory-ctl start
```

### 2. Configure Hermes MCP Client

Add to `~/.hermes/config.yaml`:

```yaml
mcp_servers:
  agentmemory:
    url: http://localhost:3111/mcp
    enabled: true
```

### 3. Verify Connection

In a new Hermes session, run:
```bash
hermes mcp test agentmemory
```

---

## Architecture Diagram

```
┌─────────────────┐     ┌──────────────────────┐     ┌─────────────────┐
│   Agent/CLI     │     │  LM Studio           │     │   SQLite        │
│                 │     │  (Port 1234)         │     │  Vector Index   │
│  agentmemory    │────▶│                      │     │  (Local DB)     │
│  MCP Server     │◀────│  Embedding Model:    │◀────│                 │
│  (Port 3111)    │     │  nomic-embed-text   │     └─────────────────┘
│                 │     └──────────────────────┘           ^
└────────┬────────┘                                        |
         │                                                 |
         ▼                                                 |
  Agent Tools:                                            |
  recall, remember,                                       |
  forget, handoff, recap                                  |
                                                         Query
                                                      Retrieve
```

---

## Quick Reference

| Environment Variable | Value | Purpose |
|---------------------|-------|---------|
| `OPENAI_BASE_URL` | `http://localhost:1234/v1` | Redirect to LM Studio |
| `OPENAI_API_KEY` | `not-needed` | Bypass auth (LM Studio local) |
| `OPENAI_EMBEDDING_MODEL` | `text-embedding-nomic-embed-text-v1.5` | Model name in LM Studio |
| `OPENAI_EMBEDDING_DIMENSIONS` | `768` | Vector dimension count |
| `EMBEDDING_PROVIDER` | `openai` | Force OpenAI provider selection |

## Commands Reference

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
```
