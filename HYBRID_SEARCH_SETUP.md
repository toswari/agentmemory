# AgentMemory Hybrid Search Setup (BM25 + Vector)

## Configuration Summary

### 1. Environment Variables Added to ~/.zshrc

```bash
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
```

### 2. Hermes Agent Configuration (Added to ~/.hermes/config.yaml)

```yaml
mcp_servers:
  agentmemory:
    url: http://localhost:3111/mcp
    enabled: true
    tools_filter: all
```

This enables Hermes to call agentmemory's MCP tools including `memory_smart_search` which uses the hybrid BM25+Vector search.

### 3. OpenCode Configuration

OpenCode connects via SDK methods dynamically. To enable agentmemory MCP tools in OpenCode sessions, ensure these environment variables are active before launching opencode:

```bash
AGENTMEMORY_TOOLS=all \
BM25_WEIGHT=0.4 \
VECTOR_WEIGHT=0.6 \
OPENAI_BASE_URL=http://localhost:1234/v1 \
EMBEDDING_PROVIDER=openai \
opencode <command>
```

Or add to your shell profile (already done in ~/.zshrc).

### 4. Starting the AgentMemory MCP Server

```bash
# Start the standalone MCP memory server
agentmemory-ctl start

# Check if it's running
agentmemory-ctl status

# Stop the server
agentmemory-ctl stop
```

The server will listen on **port 3111** (per iii-config.yaml). Logs go to `~/.local/log/agentmemory.log`.

### 5. Verification Steps

#### Verify LM Studio Embedding Endpoint
```bash
curl -X POST http://localhost:1234/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"model":"text-embedding-nomic-embed-text-v1.5","input":["test"],"encoding_format":"float"}'
```

Expected output should show a **768-dimensional vector** array.

#### Verify AgentMemory MCP Server Health
```bash
curl http://localhost:3111/health
```

Should return HTTP 200 with status information.

#### Verify Hybrid Search Weights Are Active
Check the agentmemory logs for startup confirmation:
```bash
tail ~/.local/log/agentmemory.log | grep -E "BM25|VECTOR|Embedding"
```

Expected output includes:
- `Embedding provider: openai` (or your configured provider)
- `BM25 weight: 0.4`
- `Vector weight: 0.6`
- `AGENTMEMORY_TOOLS=all loaded, enabling all tools`

#### Verify Agent Can Access Memory Tools

In Hermes session, trigger a memory search:
```bash
hermes mcp test agentmemory
```

Or use the MCP tool directly via terminal:
```bash
curl -X POST http://localhost:3111/agentmemory/search \
  -H "Content-Type: application/json" \
  -d '{"query":"test query","top_k":5}'
```

## How Hybrid Search Works

The `memory_smart_search` tool combines two retrieval methods:

1. **Vector (Semantic) Search** — Uses embeddings to find conceptually similar content
2. **BM25 (Keyword) Search** — Uses TF-IDF-like scoring for exact keyword matches

Results are weighted and ranked using:
```
FinalScore = VECTOR_WEIGHT * semantic_score + BM25_WEIGHT * keyword_score
```

With defaults of 0.6/0.4, semantic search is given more weight, but both contribute to results.

### Tuning Guidelines

| Use Case | BM25_WEIGHT | VECTOR_WEIGHT | Why |
|----------|-------------|---------------|-----|
| Technical code queries | 0.5 | 0.5 | Keywords matter for function names, APIs |
| Conceptual/explanatory queries | 0.3 | 0.7 | Semantics better for abstract questions |
| Debugging specific errors | 0.6 | 0.4 | Error messages are keyword-heavy |

## Files Modified

1. `~/projects/agentmemory/.env.example` — Added hybrid search section (Section 11)
2. `~/.hermes/config.yaml` — Added mcp_servers configuration
3. `~/.zshrc` — Added environment variables for both agents

## Next Steps After Restart

After restarting your terminal to load new ~/.zshrc:

```bash
# Reload shell config (no restart needed)
source ~/.zshrc

# Verify env vars are loaded
echo $AGENTMEMORY_TOOLS  # Should output "all"
echo $BM25_WEIGHT        # Should output "0.4"

# Start agentmemory server
agentmemory-ctl start

# Verify it's running
curl -s http://localhost:3111/health | python3 -m json.tool
```
