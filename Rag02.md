# VERITY — Complete Learning Roadmap (Phases 6–12)

> Continuation of the master learning roadmap.
> Phases 1-5 → [verity_learning_roadmap_phase_01_to_05.md](file:///Users/abhijaat/.gemini/antigravity-ide/brain/920c1b6a-2c91-4897-b3e4-59568cd39138/verity_learning_roadmap_phase_01_to_05.md)

---

# PHASE 6 — MCP (MODEL CONTEXT PROTOCOL) MASTERY

> **Note**: Verity does NOT implement MCP. This section teaches MCP from first principles because your prompt requested it and it's critical interview knowledge for AI engineers.

---

## 6.1 What is MCP?

**Definition**: Model Context Protocol (MCP) is an open standard by Anthropic that defines how AI applications (clients) communicate with external data sources and tools (servers). Think of it as "USB-C for AI" — a universal connector.

**Problem Before MCP**: Every AI tool integration was custom. Want to connect an LLM to Google Drive? Write a custom adapter. Want Slack too? Another adapter. Want GitHub? Another one. N tools × M AI apps = N×M custom integrations.

**What MCP Solves**: Standardized interface. Any MCP-compliant server works with any MCP-compliant client. N tools + M apps = N + M integrations.

## 6.2 MCP Architecture

```
┌───────────────────┐         ┌───────────────────┐
│   MCP CLIENT      │         │   MCP SERVER      │
│  (AI Application) │◄───────►│ (Tool/Data Source) │
│                   │  JSON   │                   │
│  • Claude Desktop │  -RPC   │  • File System    │
│  • VSCode AI      │  over   │  • Google Drive   │
│  • Custom App     │  STDIO/ │  • Database       │
│                   │  SSE/   │  • Web Scraper    │
│  Sends:           │  WS     │                   │
│  • tools/call     │         │  Provides:        │
│  • resources/read │         │  • Tools          │
│                   │         │  • Resources      │
│  Receives:        │         │  • Prompts        │
│  • Tool results   │         │                   │
│  • Resource data  │         │                   │
└───────────────────┘         └───────────────────┘
```

## 6.3 Core Primitives

### Tools
Functions the LLM can call. Like OpenAI function calling, but standardized.

```json
{
  "name": "search_documents",
  "description": "Search the knowledge base for relevant documents",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {"type": "string", "description": "Search query"},
      "top_k": {"type": "integer", "default": 5}
    },
    "required": ["query"]
  }
}
```

### Resources
Data the LLM can read. Like GET endpoints — read-only access to external data.

```json
{
  "uri": "file:///papers/rag_survey.pdf",
  "name": "RAG Survey Paper",
  "mimeType": "application/pdf"
}
```

### Prompts
Reusable prompt templates the server exposes.

## 6.4 Transport Layers

**STDIO** (Standard I/O): Server runs as a subprocess. Client communicates via stdin/stdout. Simplest, most common. Used by Claude Desktop.

**SSE** (Server-Sent Events): Server runs as an HTTP server. Client sends POST requests, receives SSE streams. Good for remote servers.

**WebSocket**: Bidirectional. Best for real-time, high-frequency communication.

## 6.5 Tool Invocation Lifecycle

```
1. Client sends tools/list → Server returns available tools
2. LLM decides to call a tool → Client sends tools/call
3. Server executes the tool → Returns result
4. Client feeds result back to LLM → LLM generates final answer
```

## 6.6 Pure Python MCP Server (What Verity Could Add)

```python
# verity_mcp_server.py — Hypothetical MCP server for Verity
from mcp.server import Server
from mcp.types import Tool, TextContent

server = Server("verity")

@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="search_knowledge_base",
            description="Search Verity's research paper knowledge base",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {"type": "string"},
                    "domain": {"type": "string", "enum": ["ml", "dl", "cs"]},
                    "top_k": {"type": "integer", "default": 5},
                },
                "required": ["query"],
            },
        ),
        Tool(
            name="ingest_paper",
            description="Ingest a research paper into the knowledge base",
            inputSchema={
                "type": "object",
                "properties": {
                    "url": {"type": "string"},
                    "title": {"type": "string"},
                },
                "required": ["url", "title"],
            },
        ),
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "search_knowledge_base":
        # Reuse Verity's existing HybridRetriever
        retriever = HybridRetriever()
        chunks, score = await retriever.retrieve(
            query=arguments["query"],
            domain=arguments.get("domain"),
            top_k=arguments.get("top_k", 5),
        )
        return [TextContent(text=c.text) for c in chunks]

    elif name == "ingest_paper":
        doc = await ingest_document(source=arguments["url"], title=arguments["title"])
        return [TextContent(text=f"Ingested: {doc.title} ({doc.id})")]

# Run with STDIO transport
if __name__ == "__main__":
    import mcp.server.stdio
    mcp.server.stdio.run(server)
```

## 6.7 Why Verity Doesn't Need MCP (Yet)

Verity is a self-contained application with its own frontend and API. MCP would make sense if:
- You wanted Claude Desktop to query Verity's KB directly
- You wanted Verity's retrieval to be available as a tool for other AI agents
- You were building a multi-agent system where different agents need access to different tools

---

# PHASE 7 — BACKGROUND JOBS & ASYNC PROCESSING

---

## 7.1 Why Background Jobs Exist

**The Problem**: Document ingestion (parse → chunk → embed → store) can take 5-60 seconds for a large PDF. If the API waits for completion, the user stares at a loading spinner.

**The Solution**: Accept the request immediately, queue the work, process it in a separate worker process, let the user check status later.

## 7.2 Architecture

```
┌──────────┐    1. POST /ingest/async     ┌───────────┐
│  FastAPI  │  ─────────────────────────►  │   Redis   │
│  Handler  │    2. Returns task_id        │  (Broker) │
│           │  ◄─────────────────────────  │           │
└──────────┘                               └─────┬─────┘
                                                  │
                                           3. Worker picks up
                                                  │
                                                  ▼
                                           ┌───────────┐
                                           │  Celery   │
                                           │  Worker   │
                                           │           │
                                           │ parse     │
                                           │ chunk     │
                                           │ embed     │
                                           │ store     │
                                           └─────┬─────┘
                                                  │
                                           4. Store result
                                                  │
                                                  ▼
                                           ┌───────────┐
                                           │   Redis   │
                                           │ (Backend) │
                                           └─────┬─────┘
                                                  │
    ┌──────────┐    5. GET /tasks/{id}            │
    │  FastAPI  │  ◄──────────────────────────────┘
    │  Handler  │    6. Returns status + result
    └──────────┘
```

## 7.3 Celery Configuration

### [backend/workers/celery_app.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/workers/celery_app.py)

```python
celery_app = Celery(
    "verity",
    broker=settings.redis_url,      # Redis as message broker
    backend=settings.redis_url,     # Redis as result backend
    include=["backend.workers.tasks"],  # Auto-discover tasks
)

celery_app.conf.update(
    task_serializer="json",         # Serialize task args as JSON
    result_serializer="json",       # Serialize results as JSON
    task_track_started=True,        # Track STARTED state (not just PENDING → SUCCESS)
    result_expires=3600,            # Results expire after 1 hour
)

# TLS support for managed Redis (Upstash)
if settings.redis_url.startswith("rediss://"):
    _conf["broker_use_ssl"] = {"ssl_cert_reqs": ssl.CERT_NONE}
```

**Key Concepts**:
- **Broker**: Redis acts as the message queue. Tasks are serialized and pushed to a Redis list.
- **Backend**: Redis also stores task results. The API can check `AsyncResult(task_id)` to get the status.
- **`-P solo`**: The docker-compose runs `celery worker -P solo` (single-threaded). Good for I/O-bound tasks. For CPU-bound, you'd use `-P prefork`.

## 7.4 Task Implementations

### [backend/workers/tasks.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/workers/tasks.py)

**Task 1: `ingest_document_task`** — Ingests a file from a path

```python
@celery_app.task(name="ingest_document", bind=True)
def ingest_document_task(self, source, title, domain=None, user_id=None):
    async def _run():
        from backend.db.postgres import AsyncSessionLocal
        from backend.ingestion.pipeline import ingest_document
        init_collection()
        async with AsyncSessionLocal() as db:
            doc = await ingest_document(source=source, title=title, db=db, domain=domain, user_id=user_id)
            return {"document_id": str(doc.id), "title": doc.title, "status": doc.status}
    return asyncio.run(_run())
```

**Why `asyncio.run()`**: Celery workers run synchronous code by default. The ingestion pipeline uses async SQLAlchemy. `asyncio.run()` creates a new event loop for each task execution.

**Task 2: `ingest_text_task`** — Ingests raw text (from web search)

```python
@celery_app.task(name="ingest_text", bind=True)
def ingest_text_task(self, text, title, source_url="", domain=None, user_id=None):
    async def _run():
        # Write text to temp file → run through normal pipeline
        with tempfile.NamedTemporaryFile(suffix=".txt", delete=False) as f:
            f.write(f"Source: {source_url}\n\n{text}")
            tmp_path = f.name
        try:
            async with AsyncSessionLocal() as db:
                doc = await ingest_document(source=tmp_path, title=title, db=db, ...)
        finally:
            os.unlink(tmp_path)  # Clean up temp file
```

**Design Decision**: Instead of creating a separate text ingestion path, web content is written to a temp `.txt` file and fed through the existing pipeline. This reuses all the cleaning, chunking, embedding, and storage logic.

## 7.5 Task Flow in Verity

### Explicit Async Ingest (User-triggered)
```
POST /research/ingest/async {source: "paper.pdf", title: "RAG Paper"}
    → ingest_document_task.delay(source, title, domain, user_id)
    → Returns {task_id: "abc-123", status: "queued"}

GET /research/tasks/abc-123
    → AsyncResult("abc-123").state → "SUCCESS"
    → Returns {status: "completed", result: {document_id: "..."}}
```

### Implicit Background Ingest (Web search results)
```
User query → KB miss → Web search → 5 results
    → Answer streamed immediately using raw web text
    → For each web result: ingest_text_task.delay(text, title, url, domain, user_id)
    → Celery worker: parse → chunk → embed → store in Qdrant + Postgres
    → Next query on same topic → direct KB hit
```

---

# PHASE 8 — DATABASES

---

## 8.1 PostgreSQL (Relational Database)

### Why PostgreSQL Was Chosen
- **Relational integrity**: Users → Documents → Chunks → Conversations (proper foreign keys)
- **Full-text search**: Built-in `tsvector` + GIN indexes → no separate search service needed
- **Mature async support**: `asyncpg` + SQLAlchemy async
- **Free hosting**: Neon provides free PostgreSQL with scale-to-zero

### Data Model

```
┌──────────────────┐
│      users       │
├──────────────────┤
│ id       UUID PK │
│ email    unique  │
│ hashed_password  │
│ is_active        │
│ created_at       │
└────────┬─────────┘
         │ 1:N
         ▼
┌──────────────────────────────┐
│         documents            │
├──────────────────────────────┤
│ id          UUID PK          │
│ user_id     FK → users       │←─ Per-user isolation
│ title       varchar(500)     │
│ source_path varchar(1000)    │
│ file_hash   varchar(64)      │←─ SHA-256 for dedup
│ domain      varchar(100)     │
│ status      varchar(50)      │
│ ingested_at timestamptz      │
│                              │
│ UNIQUE(user_id, file_hash)   │←─ Dedup scoped per user
└────────┬─────────────────────┘
         │ 1:N
         ▼
┌────────────────────────────┐
│          chunks            │
├────────────────────────────┤
│ id            UUID PK      │
│ document_id   FK → docs    │
│ text          TEXT          │
│ chunk_index   int          │
│ page_number   int nullable │
│ word_count    int          │
│ qdrant_id     varchar(100) │←─ Links to Qdrant vector
│ search_vector TSVECTOR     │←─ PostgreSQL full-text search
└────────────────────────────┘

┌────────────────────────────────────┐
│     conversation_history           │
├────────────────────────────────────┤
│ id          UUID PK                │
│ user_id     FK → users             │
│ session_id  varchar(100) indexed   │
│ role        varchar(20)            │←─ "user" or "assistant"
│ content     TEXT                   │
│ created_at  timestamptz            │
└────────────────────────────────────┘
```

### Full-Text Search (BM25-like)

The `chunks` table has a `search_vector` column of type `TSVECTOR` with a `GIN` index. This enables keyword-based search:

```sql
-- How sparse_retriever.py queries
SELECT c.text, ts_rank(c.search_vector, plainto_tsquery('english', :query)) AS score
FROM chunks c
WHERE c.search_vector @@ plainto_tsquery('english', :query)
ORDER BY score DESC
LIMIT :top_k
```

**Why PostgreSQL FTS over Elasticsearch**: One fewer service to deploy. For Verity's scale (thousands of chunks, not millions), PostgreSQL FTS is fast enough and eliminates the operational complexity of running Elasticsearch.

### Connection Pooling

```python
# From backend/db/postgres.py
engine = create_async_engine(
    _db_url,
    pool_size=5,          # 5 persistent connections
    max_overflow=10,      # 10 additional on burst
    pool_pre_ping=True,   # Detect dead connections (Neon scale-to-zero)
    pool_recycle=300,      # Recycle connections every 5 minutes
)
```

---

## 8.2 Qdrant (Vector Database)

### Why Qdrant Was Chosen
- **Purpose-built for vector search**: HNSW index, cosine/dot/euclidean distance
- **Payload filtering**: Filter by `user_id`, `domain`, `entities` DURING vector search (not after)
- **Cloud + local**: Free cloud tier, easy local Docker setup
- **Python client**: First-class `qdrant-client` SDK

### Data Model

```
Collection: verity_chunks
├── Vector: 384 dimensions (bge-small-en-v1.5)
├── Distance: COSINE
│
├── Payload Fields:
│   ├── document_id: string     ← Links to Postgres document
│   ├── user_id: string         ← Per-user isolation (KEYWORD index)
│   ├── chunk_index: int
│   ├── page_number: int|null
│   ├── text: string            ← Full chunk text (for display)
│   ├── domain: string          ← Domain tag (KEYWORD index)
│   ├── entities: string[]      ← Extracted entities (for graph retrieval)
│   └── chunk_type: string      ← "base" or "summary" (RAPTOR)
│
└── Payload Indexes (for fast filtering):
    ├── user_id: KEYWORD
    └── domain: KEYWORD
```

### Why Payload Indexes Matter

Without indexes, Qdrant would need to scan all vectors, compute similarity, then post-filter by user_id. With KEYWORD indexes, it pre-filters to only the user's vectors, then runs HNSW on that subset. Massive performance difference at scale.

### HNSW Index (How Vector Search Works)

**Hierarchical Navigable Small World** (HNSW) is an approximate nearest neighbor algorithm:

```
Layer 3 (sparse):     A ──────────── B
                      │               │
Layer 2:        A ──── C ──── D ──── B
                │      │      │      │
Layer 1:   A ── E ── C ── F ── D ── G ── B
                │    │    │    │    │    │
Layer 0:   A-E-H-C-I-F-J-D-K-G-L-B  (all points)
```

**Search**: Start at top layer, greedily navigate to nearest neighbor, drop to next layer, repeat. O(log n) instead of O(n).

---

## 8.3 Redis

### Three Roles in Verity

| Role | How Used | Key Pattern |
|------|----------|-------------|
| **Celery Broker** | Message queue for background tasks | `celery-task-meta-{task_id}` |
| **Celery Backend** | Store task results | `celery-task-meta-{task_id}` |
| **Rate Limiter** | Sliding window counter per user | `rl:{user_id}` |

### Rate Limiter Implementation

```python
# From backend/api/middleware/rate_limit.py
WINDOW_SECONDS = 20 * 60  # 20 minutes
MAX_REQUESTS = 20          # per user

def check_rate_limit(user_id):
    key = f"rl:{user_id}"
    pipe = _client.pipeline()
    pipe.incr(key)           # Atomic increment
    pipe.ttl(key)            # Get remaining TTL
    count, ttl = pipe.execute()

    if ttl < 0:
        _client.expire(key, WINDOW_SECONDS)  # First request: set TTL

    if count > MAX_REQUESTS:
        return False, ttl    # Blocked
    return True, 0           # Allowed
```

**Graceful degradation**: If Redis is down, the rate limiter allows all requests rather than blocking everyone. This is the correct production behavior — availability over perfect rate limiting.

---

# PHASE 9 — OBSERVABILITY

---

## 9.1 Logging Architecture

Verity uses Python's `logging` module with structured log names:

```python
# Every module creates a logger in the verity.* namespace
log = logging.getLogger("verity.agent")
log = logging.getLogger("verity.ingestion.pipeline")
log = logging.getLogger("verity.retrieval.hybrid")
```

**Log format** (from main.py):
```
16:23:45 | INFO     | verity.retrieval.hybrid | [hybrid] starting retrieval | hyde=True | domain=ml
16:23:46 | INFO     | verity.retrieval.dense  | [dense] found 20 results
16:23:46 | INFO     | verity.retrieval.hybrid | [hybrid] best_dense_score=0.723
```

**Every component logs**:
- **Input**: What was received
- **Processing**: What decisions were made
- **Output**: What was returned (counts, scores, timings)

## 9.2 Langfuse Tracing (Optional)

```python
# From backend/agents/nodes.py — only in generator_node
_langfuse = None
if settings.langfuse_public_key and settings.langfuse_secret_key:
    from langfuse import Langfuse
    _langfuse = Langfuse(public_key=..., secret_key=..., host=...)

# In generator_node:
trace = _langfuse.trace(name="rag-query", input={"query": state["query"]})
span = trace.span(name="generator", input={"chunks": len(state["chunks"])})
# ... LLM call ...
span.end(output={"answer_length": len(answer), "total_tokens": usage.total_tokens, ...})
trace.update(output={"answer_preview": answer[:200]})
```

**Current scope**: Only the generation step is traced. This is noted in the README as "planned" for full-pipeline tracing.

## 9.3 LLM-as-Judge Evaluation

### [backend/evaluation/evaluator.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/evaluation/evaluator.py)

**What**: A custom evaluation harness that uses the same LLM (Llama 3.3 70B) to judge answer quality.

**Four Metrics**:

| Metric | Question Answered | Scale |
|--------|------------------|-------|
| **Faithfulness** | Are the answer's claims supported by the context? | 0.0–1.0 |
| **Answer Relevancy** | Does the answer address the question? | 0.0–1.0 |
| **Context Recall** | Do retrieved contexts cover the reference answer? | 0.0–1.0 |
| **Context Precision** | Are retrieved contexts relevant (not noise)? | 0.0–1.0 |

**How it works**:
1. For each golden Q&A pair: run real retrieval → real generation
2. Ask the LLM to score each metric with a structured prompt
3. Parse JSON response: `{"score": 0.85, "reason": "..."}`
4. Compare baseline (RRF only) vs reranker (RRF + cross-encoder)

**Why LLM-as-judge over BLEU/ROUGE**: Traditional metrics count word overlap. They can't measure whether an answer is *faithful to its sources* or *relevant to the question*. LLM-as-judge captures semantic quality.

---

# PHASE 10 — CODE MEMORY SYSTEM

> For every major feature: Theory → Pseudocode → Actual Code → Memory Version → Interview Version

---

## 10.1 Hybrid Retrieval

### Theory
Combine semantic search (dense) with keyword search (sparse) to get both meaning-matching and exact-term-matching capabilities. Fuse results using rank-based fusion (RRF) to handle different score scales.

### Pseudocode
```
function hybrid_retrieve(query):
    hypothetical_doc = LLM("write a short answer to: {query}")
    dense_results = qdrant.search(embed(hypothetical_doc), top=20)
    sparse_results = postgres.fts(query, top=20)
    entity_results = qdrant.filter(entities=extract_entities(query), top=10)

    fused = RRF([dense, sparse, entities])
    reranked = cross_encoder.rerank(query, fused[:30])
    return reranked[:top_k]
```

### Actual Code
[hybrid_retriever.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/retrieval/hybrid_retriever.py) (105 lines)

### Memory Version (shortest possible)
```python
async def retrieve(query, db, top_k=8):
    hyde_doc = llm("answer this: " + query)
    dense = qdrant.search(embed(hyde_doc), limit=20)
    sparse = db.execute("SELECT ... WHERE search_vector @@ ...", limit=20)
    fused = rrf([dense, sparse])
    return cross_encoder.rerank(query, fused[:30])[:top_k]
```

### Interview Version
> "Our retrieval has three stages. Stage 1 fetches candidates from three sources in parallel: dense vector search using Qdrant with HyDE-expanded queries, sparse BM25 using PostgreSQL full-text search, and entity-overlap search for graph-based matching. Stage 2 fuses these using Reciprocal Rank Fusion, which is score-agnostic — it only uses ranks, solving the incompatible-score-scales problem. Stage 3 reranks the top 30 fused candidates with a cross-encoder that jointly encodes the query and each candidate, giving much more accurate relevance scores. In our evaluation, this reranking improved context recall by 41%."

---

## 10.2 Document Ingestion

### Theory
Transform raw documents into searchable, embedded chunks stored in both a vector database and a relational database.

### Memory Version
```python
async def ingest(source, title, db):
    if hash_exists(source, user_id): return  # Dedup
    pages = parser.parse(source)             # Extract text
    pages = [clean(p) for p in pages]        # Remove noise
    chunks = chunker.chunk(pages)            # Split into pieces
    chunks = [c for c in chunks if quality_ok(c)]  # Filter junk
    vectors = embedder.embed([c.text for c in chunks])  # Embed
    qdrant.upsert(points=[{id, vector, payload} for ...])  # Vector store
    db.add_all([Chunk(...) for ...])         # Relational store
    db.commit()
```

### Interview Version
> "Our ingestion pipeline has seven steps: detect file type, hash-based deduplication scoped per user, parsing with format-specific parsers (PyMuPDF for PDFs, python-docx for DOCX, trafilatura for web URLs), text cleaning that removes academic noise like DOIs and reference sections, chunking with type-appropriate strategies (recursive for PDFs, heading-based for DOCX), quality filtering that drops chunks below 30 words or with low alphabetic content, and finally batch embedding with a local ONNX model followed by dual-write to Qdrant and PostgreSQL. The dual-write is essential because dense retrieval uses Qdrant vectors while sparse retrieval uses PostgreSQL's tsvector full-text search."

---

## 10.3 LangGraph Agent Pipeline

### Memory Version
```python
workflow = StateGraph(AgentState)
workflow.add_node("planner", plan_query)
workflow.add_node("retriever", retrieve_chunks)
workflow.add_node("grader", grade_relevance)
workflow.add_node("generator", generate_answer)
workflow.add_node("hallucination", check_hallucination)

workflow.add_edge(START, "planner")
workflow.add_edge("planner", "retriever")
workflow.add_edge("retriever", "grader")
workflow.add_conditional_edges("grader", lambda s: "generator" if s["is_relevant"] else "web_search")
workflow.add_edge("generator", "hallucination")
workflow.add_edge("hallucination", END)
graph = workflow.compile()
```

### Interview Version
> "We use LangGraph to orchestrate a multi-step agentic RAG pipeline. The graph has 8 nodes: a planner that analyzes query complexity and decomposes complex queries, a retriever that runs hybrid search, a grader that uses LLM judgment to decide if retrieved chunks are relevant, a contradiction detector, a generator with streaming support, a citation verifier that checks [1][2] references against source chunks, and a hallucination checker. The graph has two conditional edges: after grading, irrelevant results route to web search instead of generation. We also have a separate latency-optimized streaming path that bypasses the LangGraph and runs retrieval + generation directly, trading quality checks for 3x faster response times."

---

## 10.4 Authentication

### Memory Version
```python
# Register
user = User(email=email, hashed_password=bcrypt.hashpw(password))
token = jwt.encode({"sub": str(user.id), "exp": now + 7_days}, SECRET, "HS256")

# Protect route
async def get_current_user(token):
    payload = jwt.decode(token, SECRET, ["HS256"])
    user = db.get(User, uuid=payload["sub"])
    return user  # All queries scoped by user.id
```

### Interview Version
> "Authentication uses bcrypt for password hashing and HS256 JWTs for stateless session tokens with a 7-day expiry. Every protected route uses a FastAPI dependency that decodes the JWT, loads the user from Postgres, and injects it into the request handler. All data access — documents, chunks, vectors, and conversations — is then scoped to that user's ID. In Qdrant, we use payload filters on the user_id field. In Postgres, every query includes a user_id WHERE clause. This provides complete tenant isolation without row-level security, which would be the next step at enterprise scale."

---

## 10.5 Streaming (SSE)

### Memory Version
```python
@router.post("/query/stream")
async def stream(request, current_user):
    chunks = await retriever.retrieve(request.query, ...)

    async def event_stream():
        for token in llm.create(messages=[...], stream=True):
            yield f"data: {json.dumps({'type': 'token', 'content': token})}\n\n"
        yield f"data: {json.dumps({'type': 'sources', 'sources': [...]})}\n\n"
        yield 'data: {"type": "done"}\n\n'

    return StreamingResponse(event_stream(), media_type="text/event-stream")
```

---

# PHASE 11 — MISSING ADVANCED CONCEPTS

> Taught from first principles even though NOT implemented in Verity.

---

## 11.1 Checkpointers & Persistence

### What
LangGraph checkpointers save the graph state after each node execution, enabling:
- Resume from where you left off
- Rewind to any previous state
- Human-in-the-loop (pause, get human input, resume)

### Types

| Store | Best For | Tradeoff |
|-------|----------|----------|
| **MemorySaver** | Development, testing | Lost on restart |
| **SqliteSaver** | Single-server apps | No horizontal scaling |
| **PostgresSaver** | Production | Requires Postgres |
| **RedisSaver** | High-throughput | Needs Redis infra |

### Why Verity Doesn't Use Them
Verity's graph runs synchronously per request — no interrupts, no human-in-the-loop. The `db` field in state (AsyncSession) isn't serializable, which would break checkpointing. Adding checkpointing would require either making the graph serializable or externalizing the DB session.

---

## 11.2 Memory Types

### Short-Term Memory (Implemented in Verity)
In-memory `deque(maxlen=10)` per session. Fast but lost on restart.

### Long-Term Memory (Implemented in Verity)
PostgreSQL `conversation_history` table. Persistent across restarts.

### Semantic Memory (Not Implemented)
Store facts extracted from conversations as embeddings. Retrieve relevant past facts based on semantic similarity to the current query.

```python
# Example: Semantic Memory Store
class SemanticMemory:
    def store(self, fact: str, user_id: str):
        vector = embed(fact)
        qdrant.upsert(collection="user_memories", vector=vector, payload={"fact": fact, "user_id": user_id})

    def recall(self, query: str, user_id: str) -> list[str]:
        vector = embed(query)
        return qdrant.search(collection="user_memories", query=vector, filter={"user_id": user_id})
```

### Episodic Memory (Not Implemented)
Store entire interaction episodes (query + context + answer + feedback) and retrieve similar past episodes. Useful for learning from past mistakes.

---

## 11.3 Agent Architectures

### ReAct (Reasoning + Acting)
```
Think: I need to find papers about RAG
Act: search_knowledge_base("RAG approaches")
Observe: Found 5 chunks about RAG
Think: I have enough context to answer
Act: generate_answer(chunks)
```

### Reflection Agents
After generating an answer, the agent critiques itself:
```
Generate: "RAG uses vector search..."
Reflect: "My answer doesn't mention sparse retrieval, which is important"
Regenerate: "RAG uses both vector (dense) and keyword (sparse) search..."
```

### Supervisor Agents
A supervisor LLM routes tasks to specialized worker agents:
```
Supervisor: "This is a comparison question → route to ComparisonAgent"
ComparisonAgent: runs dedicated comparison workflow
Supervisor: collects results, synthesizes final answer
```

### Multi-Agent Systems
Multiple agents with different roles collaborate:
```
Researcher Agent: retrieves information
Critic Agent: checks for errors
Writer Agent: synthesizes final answer
Each agent has its own LLM, tools, and state
```

---

## 11.4 Guardrails

### Prompt Injection Defense
```python
# Example: Sanitize user input before it reaches the LLM
def sanitize_query(query: str) -> str:
    # Block attempts to override system prompt
    injection_patterns = ["ignore previous", "you are now", "system:", "forget your"]
    for pattern in injection_patterns:
        if pattern.lower() in query.lower():
            raise ValueError("Potential prompt injection detected")
    return query
```

### Output Guardrails
```python
# Example: Ensure answer doesn't leak sensitive data
def check_output(answer: str) -> str:
    # Remove any leaked API keys, emails, etc.
    answer = re.sub(r'sk-[a-zA-Z0-9]{20,}', '[REDACTED]', answer)
    answer = re.sub(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b', '[REDACTED]', answer)
    return answer
```

---

## 11.5 Streaming Deep Dive

### Server-Sent Events (SSE) — What Verity Uses

```
Client                          Server
  |---- POST /query/stream ------>|
  |                               |
  |<--- data: {"type":"token",    |
  |      "content":"RAG"}         |
  |<--- data: {"type":"token",    |
  |      "content":" is"}         |
  |<--- data: {"type":"token",    |
  |      "content":" a"}          |
  |<--- data: {"type":"sources",  |
  |      "sources":[...]}         |
  |<--- data: {"type":"done"}     |
  |                               |
```

**Why SSE over WebSockets**: SSE is simpler (HTTP, auto-reconnect, one-directional), and LLM streaming is inherently one-directional (server → client). WebSockets add unnecessary complexity.

---

## 11.6 Cost Optimization

### Verity's Cost Decisions

| Decision | Impact |
|----------|--------|
| Free-tier LLM models (`:free` suffix) | $0/month LLM cost |
| Local ONNX embeddings (no API) | $0/month embedding cost |
| Local ONNX reranker (no API) | $0/month reranking cost |
| HyDE uses `fast_llm_model` (Gemma 3 4B) | Cheaper than main model |
| Stream path skips 5 LLM calls | 6x fewer tokens per query |
| Hash dedup prevents re-ingestion | Saves embedding compute |
| Entity extraction OFF by default | Saves 1 LLM call per chunk |

### Production Cost Formula
```
Cost per query ≈ (embedding_cost × 1) + (LLM_cost × num_calls)

Full agentic path:  1 embed + 6 LLM calls ≈ $0.002-0.01 (paid models)
Stream path:        1 embed + 1 LLM call  ≈ $0.0005-0.002

Ingestion cost per document:
  = chunks × embedding_cost + (if entities: chunks × LLM_cost)
  = 100 chunks × $0.00001 = $0.001 without entities
  = 100 chunks × $0.001   = $0.10  with entities
```

---

# PHASE 12 — REBUILD PROJECT FROM SCRATCH

---

## Step 1: Project Setup

```bash
mkdir verity && cd verity
python -m venv venv && source venv/bin/activate

# Core structure
mkdir -p backend/{agents,api/{routes,schemas,dependencies,middleware},core,db/models,ingestion/{parsers,chunking,cleaning,embedding,graph,raptor,multimodal},retrieval,memory,workers,prompts,evaluation}
mkdir -p frontend scripts tests/{unit,integration,eval} migrations/versions data Architecture

# Install core dependencies
pip install fastapi uvicorn pydantic pydantic-settings
pip install sqlalchemy asyncpg alembic  # Database
pip install qdrant-client fastembed      # Vector DB + embeddings
pip install openai                        # LLM client
pip install celery redis                  # Background jobs
pip install pymupdf python-docx trafilatura  # Parsers
pip install langgraph langchain-core langchain-text-splitters  # Agent orchestration
pip install pyjwt bcrypt                  # Auth
```

## Step 2: Build Config & LLM Client
```
backend/core/config.py     ← pydantic-settings, load .env
backend/core/llm.py        ← OpenAI client pointing at OpenRouter
backend/core/security.py   ← bcrypt + JWT functions
```

## Step 3: Build Database Layer
```
backend/db/postgres.py     ← SQLAlchemy async engine + session factory
backend/db/qdrant_client.py ← Qdrant client + collection init
backend/db/models/user.py   ← User model
backend/db/models/document.py ← Document model with file_hash
backend/db/models/chunk.py   ← Chunk model with tsvector
backend/db/models/conversation.py ← ConversationHistory model
```

## Step 4: Build Parsers
```
backend/ingestion/parsers/base.py  ← ABC: parse(source) → list[ParsedPage]
backend/ingestion/parsers/pdf_parser.py  ← PyMuPDF
backend/ingestion/parsers/docx_parser.py ← python-docx with heading splitting
backend/ingestion/parsers/web_parser.py  ← trafilatura
backend/ingestion/parsers/txt_parser.py  ← simple file read
```

## Step 5: Build Text Cleaner
```
backend/ingestion/cleaning/text_cleaner.py ← Regex patterns for noise removal
```

## Step 6: Build Chunkers
```
backend/ingestion/chunking/base.py          ← ABC: chunk(pages) → list[TextChunk]
backend/ingestion/chunking/recursive_chunker.py ← Default: 800 chars, 150 overlap
backend/ingestion/chunking/heading_chunker.py   ← For DOCX
backend/ingestion/chunking/markdown_chunker.py  ← For Markdown
backend/ingestion/chunking/factory.py           ← doc_type → chunker
```

## Step 7: Build Embedding Service
```
backend/ingestion/embedding/embedding_service.py ← fastembed TextEmbedding
```

## Step 8: Build Ingestion Pipeline
```
backend/ingestion/pipeline.py ← The 7-step pipeline orchestrator
```

## Step 9: Build Retrieval Layer
```
backend/retrieval/dense_retriever.py    ← Qdrant cosine search
backend/retrieval/sparse_retriever.py   ← PostgreSQL FTS
backend/retrieval/hyde.py               ← Hypothetical document generation
backend/retrieval/graph_expander.py     ← Entity-based retrieval
backend/retrieval/hybrid_retriever.py   ← Orchestrates all + RRF + rerank
backend/retrieval/reranker.py           ← Cross-encoder via fastembed
backend/retrieval/context_assembler.py  ← Format chunks for LLM
backend/retrieval/web_searcher.py       ← Tavily / DuckDuckGo fallback
```

## Step 10: Build Agent Pipeline
```
backend/agents/state.py  ← AgentState TypedDict
backend/agents/nodes.py  ← All 8+ node functions
backend/agents/graph.py  ← StateGraph construction
backend/prompts/report.py ← Generator prompt template
```

## Step 11: Build API Layer
```
backend/api/dependencies/auth.py    ← get_current_user dependency
backend/api/middleware/rate_limit.py ← Redis rate limiter
backend/api/schemas/auth.py         ← Request/response models
backend/api/schemas/research.py     ← Query/Ingest models
backend/api/routes/auth.py          ← /register, /login, /me
backend/api/routes/research.py      ← /upload, /ingest, /query, /stream, /sessions
```

## Step 12: Build Background Workers
```
backend/workers/celery_app.py  ← Celery config with Redis
backend/workers/tasks.py       ← ingest_document_task, ingest_text_task
```

### Build Order Rationale
```
Config → DB → Parsers → Cleaners → Chunkers → Embedder → Pipeline
                                                            ↓
                                             Retrievers → Reranker → Agents
                                                            ↓
                                                     API Routes → Workers
```

Each layer depends only on layers built before it. This is the correct dependency order.

### Common Mistakes to Avoid

| Mistake | Why It's Wrong | Verity's Solution |
|---------|---------------|-------------------|
| Using LangChain for everything | Over-abstraction, hard to debug | Raw OpenAI client + custom retrievers |
| Chunking before cleaning | Noise ends up in chunks | Clean pages → then chunk |
| Single retrieval method | Dense misses keywords, sparse misses meaning | Hybrid (dense + sparse + graph) |
| No deduplication | Same file ingested twice = duplicate chunks | SHA-256 hash per user |
| Blocking API for ingestion | User waits 30+ seconds | Celery async tasks |
| Same path for speed and quality | Can't optimize both | Two paths: stream (fast) vs full (quality) |
| Global database state | Tight coupling | DB session passed through state/dependency injection |
| No user isolation | Data leaks between users | user_id on every table + every query filter |
