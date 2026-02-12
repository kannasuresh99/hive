# Hive Trace Debugger

Production-grade debugging and analysis tool for Hive agent execution traces. Uses RAG (Retrieval-Augmented Generation) to provide intelligent insights about agent behavior, failures, and performance.

## Features

- 🔍 **Semantic Search**: Find similar failures and execution patterns
- 🤖 **AI-Powered Analysis**: Query traces using natural language
- 🌐 **Multi-Provider LLM Support**: Works with **OpenAI**, **Anthropic (Claude)**, and **Google (Gemini)**
- 📊 **Pattern Detection**: Automatically identify failure clusters, performance issues, retry loops
- 💾 **Persistent Storage**: File-based indexes for fast querying
- ⚡ **Fast**: FAISS vector search + batch embedding
- 🧪 **Well-Tested**: >95% test coverage, comprehensive end-to-end tests

## LLM Provider Configuration

The debugger supports **OpenAI**, **Anthropic (Claude)**, and **Google (Gemini)** via LangChain.

### Auto-Detection

Set your API key and the provider will be auto-detected:

```bash
# Use OpenAI (GPT-4, GPT-4-Turbo, etc.)
export OPENAI_API_KEY="sk-..."

# Or use Anthropic (Claude 3.5 Sonnet, Opus, etc.)
export ANTHROPIC_API_KEY="sk-ant-..."

# Or use Google (Gemini 1.5 Pro, Flash, etc.)
export GOOGLE_API_KEY="..."
```

### Explicit Provider Selection

Override auto-detection using environment variable or CLI flags:

```bash
# Via environment variable
export HIVE_LLM_PROVIDER="openai"  # or "anthropic" or "google"

# Via CLI flag
python -m framework.debugging.cli query ... --provider openai
python -m framework.debugging.cli query ... --provider anthropic --model claude-3-opus-20240229
python -m framework.debugging.cli query ... --provider google --model gemini-1.5-flash
```

### Supported Models

| Provider | Default Model | Other Options |
|----------|--------------|---------------|
| **OpenAI** | `gpt-4` | `gpt-4-turbo`, `gpt-4o`, `gpt-3.5-turbo` |
| **Anthropic** | `claude-3-5-sonnet-20241022` | `claude-3-opus-20240229`, `claude-3-haiku-20240307` |
| **Google** | `gemini-1.5-pro` | `gemini-1.5-flash`, `gemini-1.0-pro` |

## Quick Start

### 1. Index Your Traces

Scan session directories and build searchable index:

```bash
python -m framework.debugging.cli index ~/.hive/agents/sales_agent
```

Output:
```
Indexing traces from: /Users/you/.hive/agents/sales_agent
Existing index: 0 traces

✓ Indexing complete!
  Indexed: 15
  Skipped: 2
  Errors: 0
  Total traces: 15
```

### 2. Build Vector Index (for semantic search)

Generate embeddings and build FAISS index:

```bash
python -m framework.debugging.cli build-vector-index ~/.hive/agents/sales_agent
```

Output:
```
Building vector index for: /Users/you/.hive/agents/sales_agent
Found 15 traces
Initializing embedder (downloading model if needed)...
Generating embeddings...
[████████████████████████████████] 15/15
Saving vector index...

✓ Vector index built: 15 traces
```

### 3. Query with Natural Language

Ask questions about your traces:

```bash
python -m framework.debugging.cli query ~/.hive/agents/sales_agent \
  "Why did my agent fail in the research node?"
```

Output:
```
Querying: Why did my agent fail in the research node?

Answer:
--------------------------------------------------------------------------------
The agent failed in the research node due to timeouts in the web_search tool.
Analysis of 5 similar failures shows:
- All failures occurred during web API calls
- Average retry count: 6 attempts
- Root cause: External API rate limiting

Recommendation: Implement exponential backoff or switch to cached data source.
--------------------------------------------------------------------------------

Retrieved Traces (5):
1. session_20260212_120000_abc
   Status: failure
   Agent: sales_agent
   Latency: 8000ms
   Tokens: 5000
   Error: Timeout in web_search
...
```

### 4. Analyze Patterns

Identify common patterns across runs:

```bash
python -m framework.debugging.cli analyze ~/.hive/agents/sales_agent --pattern failures
```

Pattern types:
- `failures` - Common failure patterns and root causes
- `performance` - High latency, token usage, slow nodes
- `retry_patterns` - Retry loops and escalation patterns
- `error_clusters` - Group similar errors together

### 5. View Statistics

Get overview of your traces:

```bash
python -m framework.debugging.cli stats ~/.hive/agents/sales_agent
```

Output:
```
Trace Statistics for: /Users/you/.hive/agents/sales_agent
================================================================================
Total Traces: 15
  Success: 10 (66.7%)
  Failure: 4 (26.7%)
  Degraded: 1 (6.7%)

Average Latency: 5234ms
Average Tokens: 3421

Most Common Errors:
  3x: Timeout in web_search
  1x: Missing required output 'twitter_handle'

Vector Index: 15 traces indexed
```

## Programmatic Usage

### Python API

```python
from pathlib import Path
from framework.debugging import (
    TraceIndexer,
    IndexStore,
    TraceEmbedder,
    TraceVectorStore,
    TraceRAG
)

# 1. Index traces
agent_path = Path.home() / ".hive" / "agents" / "sales_agent"
indexer = TraceIndexer(agent_path)
store = IndexStore(base_path=agent_path)

stats = await indexer.index_all_sessions(store)
await store.save()

print(f"Indexed {stats['indexed']} traces")

# 2. Build vector index
embedder = TraceEmbedder()
vector_store = TraceVectorStore(storage_path=agent_path / ".vector_index")
await vector_store.initialize()

traces = store.list_all()
embeddings = await embedder.embed_traces(traces)
await vector_store.add_traces(traces, embeddings)
await vector_store.save()

# 3. Query with RAG
rag = TraceRAG(embedder, vector_store)

result = await rag.query("What caused the failures?", k=5)
print(result["answer"])

for trace in result["traces"]:
    print(f"- {trace.run_id}: {trace.status}")

# 4. Analyze patterns
analysis = await rag.analyze_pattern("failures", limit=10)
print(analysis)

# 5. Find similar failures
failed_trace = [t for t in traces if t.status == "failure"][0]
similar = await rag.find_similar_failures(failed_trace, k=5)

for trace, distance in similar:
    print(f"- {trace.run_id} (similarity: {1/(1+distance):.2f})")
```

## Architecture

### Components

```
┌─────────────────────────────────────────────────────────────┐
│                     User Interface                           │
│  CLI Commands │ Python API │ Future: Web UI, MCP Tools      │
└───────────────────┬─────────────────────────────────────────┘
                    │
        ┌───────────▼──────────┬──────────────┬───────────────┐
        │                      │              │               │
   ┌────▼─────┐     ┌─────────▼────┐  ┌─────▼──────┐  ┌────▼────┐
   │ Indexer  │     │  Embedder    │  │  Vector    │  │   RAG   │
   │          │     │              │  │   Store    │  │  Engine │
   │ - Scans  │     │ - Trace →    │  │            │  │         │
   │   L1/L2/ │     │   Document   │  │ - FAISS    │  │ - Query │
   │   L3     │     │ - Embedding  │  │ - Search   │  │ - LLM   │
   │ - Builds │     │   Generation │  │ - Persist  │  │ - Insights
   │   Index  │     │              │  │            │  │         │
   └────┬─────┘     └──────────────┘  └────────────┘  └─────────┘
        │                      │              │               │
        └───────────┬──────────┴──────────────┴───────────────┘
                    │
        ┌───────────▼───────────────────────────────────────────┐
        │              Persistent Storage                       │
        │  .trace_index.json │ .vector_index/ │ FAISS index    │
        └───────────────────────────────────────────────────────┘
```

### Data Flow

1. **Indexing**: Session logs → TraceIndexer → TraceIndex objects → IndexStore
2. **Embedding**: TraceIndex → TraceEmbedder → Embeddings → TraceVectorStore
3. **Querying**: Question → Embedding → Vector Search → Context → LLM → Answer

### Storage

```
~/.hive/agents/{agent_name}/
├── sessions/                    # Session logs (L1/L2/L3)
│   └── session_*/logs/
│       ├── summary.json         # L1: Run outcome
│       ├── details.jsonl        # L2: Per-node results
│       └── tool_logs.jsonl      # L3: Step-by-step execution
├── .trace_index.json            # Searchable trace metadata
└── .vector_index/               # FAISS vector store
    ├── faiss.index              # Vector index
    ├── metadata.json            # Trace metadata
    └── index_mapping.json       # Index → run_id mapping
```

## Advanced Usage

### Custom Embeddings Provider

Use your own embeddings model:

```python
from langchain_openai import OpenAIEmbeddings

# Use OpenAI embeddings instead of HuggingFace
embeddings = OpenAIEmbeddings(model="text-embedding-3-large")
embedder = TraceEmbedder(embeddings_provider=embeddings)
```

### Custom LLM

Use different LLM for RAG:

```python
from langchain_anthropic import ChatAnthropic

llm = ChatAnthropic(model="claude-opus-4-6", temperature=0)
rag = TraceRAG(embedder, vector_store, llm=llm)
```

### Filter Traces

Query specific subsets:

```python
# Find all failures
failures = [t for t in store.list_all() if t.status == "failure"]

# Find high-latency runs
slow_runs = [t for t in store.list_all() if t.total_latency_ms > 10000]

# Find runs with errors in specific node
research_errors = [
    t for t in store.list_all()
    if t.failed_node_id == "research"
]
```

### Incremental Updates

Add new sessions without rebuilding entire index:

```python
# Index new sessions
stats = await indexer.index_all_sessions(store)
print(f"Added {stats['indexed']} new traces")

# Add to vector store
new_traces = [store.get(run_id) for run_id in new_run_ids]
new_embeddings = await embedder.embed_traces(new_traces)
await vector_store.add_traces(new_traces, new_embeddings)

await store.save()
await vector_store.save()
```

## Performance

### Indexing
- **Speed**: ~100 sessions/second (depends on log size)
- **Memory**: ~10MB per 1000 traces
- **Storage**: ~50KB per trace (index + embeddings)

### Querying
- **Latency**:
  - Vector search: ~10-50ms (FAISS)
  - LLM generation: ~2-5s (Claude API)
- **Throughput**: Limited by LLM API rate limits

### Embedding
- **Model**: HuggingFace all-MiniLM-L6-v2 (local, 384 dimensions)
- **Speed**: ~1000 traces/minute on CPU
- **Alternative**: Use OpenAI embeddings for faster performance

## Troubleshooting

### "No traces found"

```bash
# Check if sessions exist
ls ~/.hive/agents/sales_agent/sessions/

# Run index command
python -m framework.debugging.cli index ~/.hive/agents/sales_agent
```

### "Vector index not found"

```bash
# Build vector index first
python -m framework.debugging.cli build-vector-index ~/.hive/agents/sales_agent
```

### Slow embedding generation

```python
# Use OpenAI embeddings for faster performance
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
embedder = TraceEmbedder(embeddings_provider=embeddings)
```

### Out of memory

```python
# Process traces in smaller batches
batch_size = 10
for i in range(0, len(traces), batch_size):
    batch = traces[i:i + batch_size]
    embeddings = await embedder.embed_traces(batch)
    await vector_store.add_traces(batch, embeddings)
```

## Testing

Run full test suite:

```bash
cd core
pytest framework/debugging/tests/ -v --cov=framework.debugging
```

Run end-to-end workflow test:

```bash
pytest framework/debugging/tests/test_e2e_workflow.py -v
```

## Contributing

The debugging infrastructure follows Hive patterns:
- Pydantic models for type safety
- Async I/O with `asyncio.to_thread()`
- File-based storage (no databases)
- Comprehensive test coverage (>95%)
- Non-fatal error handling

See existing code for examples.

## Future Enhancements

- [ ] Web UI for visual trace exploration
- [ ] MCP tools integration
- [ ] Real-time indexing (watch for new sessions)
- [ ] Multi-agent comparison
- [ ] Custom pattern detectors
- [ ] Integration with /hive-debugger skill
- [ ] Export to external analysis tools

## License

Part of the Hive framework. See main repository for license details.
