# RAG-CLI v2.0

**Local Retrieval-Augmented Generation system for Claude Code with Multi-Agent Framework integration.**

A production-ready Claude Code plugin that combines ChromaDB vector embeddings, hybrid BM25 + semantic search, and a Multi-Agent Framework (MAF) for context-aware development assistance — with all document processing happening entirely on your machine.

---

## Project Status

| Item | Detail |
|------|--------|
| Version | 2.0.0 |
| Python | 3.8 – 3.13 |
| Status | Beta (known limitations in [KNOWN_ISSUES.md](KNOWN_ISSUES.md)) |
| License | MIT |

---

## Features

- **Hybrid search** — 70 % semantic vector search + 30 % BM25 keyword matching, tunable via `config/default.yaml`
- **Cross-encoder reranking** — `cross-encoder/ms-marco-MiniLM-L-6-v2` for precision after initial retrieval
- **ChromaDB vector store** — persistent HNSW-indexed storage; auto-upgrades index strategy (Flat → HNSW at 2 000 vectors → IVF at 1 M vectors)
- **HyDE (Hypothetical Document Embeddings)** — generates a hypothetical answer before embedding the query; improves retrieval accuracy 10–15 % on technical queries
- **Semantic cache** — TTL-based cache with LRU eviction; avoids re-embedding identical or near-identical queries
- **Multi-Agent Framework (MAF)** — routes queries to the right agent: RAG-only, code analysis, parallel RAG + MAF, or decomposed multi-part
- **MCP server** — exposes retrieval as a Model Context Protocol tool for Claude Code
- **Background file watcher** — debounced auto-indexing via `watchdog`
- **Real-time monitoring** — TCP server on port 9999 + Flask web dashboard
- **Multi-format ingestion** — Markdown, PDF, DOCX, HTML, TXT (max 10 MB per file)
- **Complete privacy** — no document content leaves your machine

---

## Architecture

```
Claude Code
    │
    ├── Hooks (UserPromptSubmit, SessionStart/End, ErrorHandler …)
    │       └── user-prompt-submit.py   ← main entry point
    │
    ├── MCP Server  (src/rag_cli_plugin/mcp/unified_server.py)
    │
    └── Slash Commands  (/search, /rag-enable, /rag-disable, /rag-project)

RAG Core  (src/rag_cli/core/)
    ├── document_processor.py   — chunking (500 tokens / 100 overlap)
    ├── embeddings.py           — sentence-transformers/all-MiniLM-L6-v2 (384-dim)
    ├── vector_store.py         — ChromaDB + HNSW
    ├── retrieval_pipeline.py   — async hybrid search + HyDE + reranking
    ├── semantic_cache.py       — TTL / LRU semantic cache
    ├── query_classifier.py     — intent detection for adaptive weights
    ├── hyde.py                 — Hypothetical Document Embeddings
    └── claude_integration.py  — Anthropic API (standalone mode)

MAF  (src/rag_cli/agents/maf/)
    ├── core/orchestrator.py    — query routing
    ├── agents/developer.py     — code tasks
    ├── agents/debugger.py      — error analysis
    ├── agents/architect.py     — design decisions
    └── agents/documenter.py    — documentation tasks

Plugin Services  (src/rag_cli_plugin/services/)
    ├── tcp_server.py           — metrics & status (port 9999)
    ├── dashboard.py            — web dashboard
    └── service_manager.py      — process lifecycle
```

---

## Installation

### Prerequisites

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| Python | 3.8 | 3.11+ |
| RAM | 4 GB | 8 GB |
| Disk | 2 GB | 5 GB |
| Claude Code | Latest | Latest |

### Option 1 — Claude Code Marketplace (recommended)

```bash
# Inside Claude Code
/plugin install rag-cli
```

Restart Claude Code. The plugin activates automatically with no further configuration.

### Option 2 — Install from source

```bash
git clone https://github.com/Sharmilab299/rag-cli.git
cd rag-cli

python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Verify
python -c "from rag_cli.core import embeddings; print('OK')"
```

### Option 3 — Editable / development install

```bash
git clone https://github.com/Sharmilab299/rag-cli.git
cd rag-cli

python -m venv venv
source venv/bin/activate

pip install -e ".[dev]"

python scripts/configure_mcp.py   # generates .mcp.json with absolute paths (gitignored)

pytest tests/
```

---

## Quick Start

### 1. Index your documents

```bash
# As a Claude Code plugin — indexes the current project
/rag-project

# Or from the terminal
rag-index --input /path/to/docs
```

Supported formats: `.md`, `.txt`, `.docx`, `.html` — and `.pdf` once the `.pd` typo in `config.py` is patched (see [KNOWN_ISSUES](KNOWN_ISSUES.md)).

### 2. Search

```bash
# Slash command inside Claude Code
/search "How do I configure authentication?"

# CLI
rag-retrieve "How do I configure authentication?"
```

### 3. Enable automatic context injection

```bash
/rag-enable    # all prompts are silently enhanced with retrieved context
/rag-disable   # turn it off
```

---

## Configuration

Edit `config/default.yaml`:

```yaml
embeddings:
  model_name: sentence-transformers/all-MiniLM-L6-v2
  dimensions: 384
  batch_size: 32
  device: cpu                        # cpu | cuda | mps

vector_store:
  backend: chromadb
  save_path: data/vectors/chroma_db

retrieval:
  vector_weight: 0.7                 # must sum to 1.0 with keyword_weight
  keyword_weight: 0.3
  initial_candidates: 10
  final_results: 5
  use_reranker: true
  reranker_model: cross-encoder/ms-marco-MiniLM-L-6-v2
  min_score_threshold: 0.5
  cache_enabled: true
  cache_ttl_seconds: 3600

# Standalone mode only
claude:
  model: claude-haiku-4-5-20251001
  max_tokens: 4096
  api_key_env: ANTHROPIC_API_KEY
```

All tuneable numbers are also centralised in `src/rag_cli/core/constants.py`.

### Environment variables

| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_API_KEY` | Required for standalone mode only |
| `RAG_CLI_MODE` | `claude_code` / `standalone` / `hybrid` |
| `RAG_CLI_LOG_LEVEL` | `DEBUG` / `INFO` / `WARNING` / `ERROR` |
| `TAVILY_API_KEY` | Optional — enables web fallback via Tavily |
| `GITHUB_TOKEN` | Optional — enables GitHub source connector |

---

## Operation Modes

| Mode | API key needed | Best for |
|------|---------------|---------|
| `claude_code` | No | Running as a Claude Code plugin |
| `standalone` | Yes | Testing outside Claude Code |
| `hybrid` | Optional | Auto-detects environment |

```bash
export RAG_CLI_MODE=claude_code
```

---

## Slash Commands

| Command | Description |
|---------|-------------|
| `/search [query]` | Search indexed documents |
| `/rag-enable` | Auto-enhance all prompts with RAG context |
| `/rag-disable` | Turn off auto-enhancement |
| `/rag-project` | Index the current project |
| `/update-rag` | Sync plugin files after an update |

---

## Multi-Agent Framework

The MAF orchestrator selects a strategy automatically based on query intent:

| Strategy | When used |
|----------|----------|
| RAG only | Documentation / how-to questions |
| MAF only | Pure code analysis, debugging |
| Parallel RAG + MAF | Questions mixing docs and code |
| Decomposed | Multi-part queries split into sub-tasks |

Available agents: `developer`, `debugger`, `architect`, `reviewer`, `optimizer`, `documenter`, `tester`.

---

## Monitoring

The TCP server runs on port 9999 and accepts text commands:

```bash
# Python
import socket, json
with socket.socket() as s:
    s.connect(("localhost", 9999))
    s.send(b"STATUS")
    print(json.loads(s.recv(4096)))

# PowerShell
./scripts/monitor.ps1 -Command STATUS
```

Supported commands: `STATUS`, `METRICS`, `LOGS`, `HEALTH`.

A Flask web dashboard is available at `http://localhost:8080` when the monitoring service is running:

```bash
python -m rag_cli_plugin.services
```

---

## Python API

```python
# Index documents
from rag_cli.core.document_processor import get_document_processor
from rag_cli.core.embeddings import get_embedding_generator
from rag_cli.core.vector_store import get_vector_store

processor = get_document_processor()
chunks    = processor.process_directory("data/documents")

generator = get_embedding_generator()
embeddings = generator.encode_documents([c.text for c in chunks])

store = get_vector_store()
store.add(embeddings, [c.text for c in chunks], metadata=[c.metadata for c in chunks])

# Retrieve
from rag_cli.core.retrieval_pipeline import get_retriever

retriever = get_retriever()
results   = retriever.retrieve("your query here", top_k=5)

for r in results:
    print(r.score, r.source, r.text[:120])
```

---

## Performance

| Operation | Target | Typical |
|-----------|--------|---------|
| Vector search | < 100 ms | ~45 ms |
| Full pipeline (no reranker) | < 1 s | ~300 ms |
| Full pipeline (with reranker) | < 5 s | ~3.2 s |
| Embedding generation (100 docs) | < 500 ms | ~200 ms |

**Tuning tips:**

- Reduce `final_results` and set `use_reranker: false` for fastest responses
- Use `device: cuda` or `device: mps` if a GPU is available
- Increase `vector_weight` toward 1.0 for highly semantic corpora
- Increase `keyword_weight` for error messages and exact-match queries

---

## Testing

```bash
pytest                                    # all tests
pytest --cov=src --cov-report=html        # with coverage
pytest tests/test_core.py::TestEmbeddings # single test
```

---

## Known Issues

See [KNOWN_ISSUES.md](KNOWN_ISSUES.md) for the current list. Key items:

- **PDF typo** (`config.py:51`): `.pd` instead of `.pdf` — PDFs are silently skipped until patched
- **Inline citations disabled**: the `PostToolUse` hook (`response-post.py`) is off due to a JSON-parsing bug in the Claude Code plugin framework
- **Windows marketplace install**: fixed in the current commit via `CLAUDE_LIFECYCLE_HOOK` environment variable and explicit resource cleanup

---

## Project Structure

```
rag-cli/
├── src/
│   ├── rag_cli/                 # Core library
│   │   ├── core/                # RAG pipeline, vector store, embeddings
│   │   ├── agents/              # MAF agents and orchestrator
│   │   ├── integrations/        # Tavily, arXiv, MAF connectors
│   │   └── cli/                 # index / retrieve CLI entry points
│   └── rag_cli_plugin/          # Claude Code plugin
│       ├── hooks/               # Event hooks
│       ├── mcp/                 # MCP server
│       ├── commands/            # Slash command definitions
│       ├── services/            # TCP server, dashboard, service manager
│       └── lifecycle/           # installer.py, updater.py
├── config/                      # default.yaml, schema, per-run overrides
├── scripts/                     # setup, verify, migrate utilities
├── tests/                       # pytest suites
├── data/                        # vectors/ and documents/ (gitignored)
└── .claude-plugin/              # hooks.json, lifecycle.json
```

---

## Security

- Never commit `.env` files — they are gitignored by default
- Set `chmod 600 .env` on Unix to restrict key access
- Subprocess calls use `shell=False` (list form) to prevent command injection — verify this in `claude_cli_unified.py` if you are running an unpatched version
- Sensitive environment variables (`ANTHROPIC_API_KEY`, `TAVILY_API_KEY`) are stripped before spawning subprocesses

See [SECURITY.md](SECURITY.md) for full details.

---

## Contributing

1. Fork the repository: [github.com/Sharmilab299/rag-cli](https://github.com/Sharmilab299/rag-cli)
2. Create a feature branch
3. Add tests for new functionality
4. Run `pytest` and `black src/`
5. Open a pull request

Please read [CONTRIBUTING.md](CONTRIBUTING.md) before submitting.

---

## Dependencies

Core dependencies (see `requirements.txt` for pinned versions):

| Package | Purpose |
|---------|---------|
| `chromadb` | Vector store with HNSW indexing |
| `sentence-transformers` | Embedding generation |
| `bm25s` | Keyword search |
| `anthropic` | Claude API (standalone mode) |
| `langchain` / `langchain-community` | Document parsing |
| `pypdf`, `python-docx`, `beautifulsoup4` | Format-specific extractors |
| `structlog` | Structured logging |
| `flask` + `waitress` | Web dashboard |
| `prometheus-client` | Metrics export |

---

## Acknowledgements

- [Sentence Transformers](https://www.sbert.net/) — embedding models
- [ChromaDB](https://www.trychroma.com/) — vector database with HNSW indexing
- [Anthropic](https://www.anthropic.com/) — Claude API
- [LangChain](https://langchain.com/) — document loaders and text splitters

---

## License

MIT — see [LICENSE](LICENSE) for details.

---

## Links

- **Repository**: [github.com/Sharmilab299/rag-cli](https://github.com/Sharmilab299/rag-cli)
- **Bug reports**: [github.com/Sharmilab299/rag-cli/issues](https://github.com/Sharmilab299/rag-cli/issues)
- **Known issues**: [KNOWN_ISSUES.md](KNOWN_ISSUES.md)
- **Security policy**: [SECURITY.md](SECURITY.md)
