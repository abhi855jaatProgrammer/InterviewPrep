# VERITY — Complete Learning Roadmap (Phases 1–5)

> **Goal**: Master every concept, rebuild the project from scratch, and explain it all in interviews.
> Every section maps theory → Verity's actual code → memory version → interview explanation.

---

# PHASE 1 — REPOSITORY ANALYSIS

---

## 1.1 High-Level Overview

### What Problem Verity Solves

**The Problem**: A researcher has 20+ papers on a topic (e.g., RAG techniques). Reading each paper takes 2-3 hours. They need synthesized answers across ALL papers, not one at a time.

**The Solution**: Verity ingests research papers (PDF, DOCX, web URLs), breaks them into chunks, embeds them into a vector database, and lets you ask natural language questions. It synthesizes answers from ALL ingested papers with inline citations `[1][2]`.

**Who Uses It**: AI researchers, graduate students, engineers building AI systems — anyone who needs to synthesize knowledge from a corpus of documents.

**Why Built**: To learn and demonstrate every layer of a production-grade RAG system — from document parsing to agentic pipelines — not just use a framework.

**Business Objective**: Enable research question answering with verifiable, cited answers from a personal knowledge base.

**Technical Objective**: Production-grade implementation of:
- Hybrid retrieval (dense + sparse + graph, fused with RRF, reranked with cross-encoder)
- Agentic RAG with LangGraph (planner → grader → contradiction detection → hallucination checking)
- Self-improving knowledge base (web search results auto-ingested into KB)
- Multi-user isolation (every piece of data scoped to its owner)

---

## 1.2 Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                         USER (Browser)                               │
│                    Next.js 16 + Tailwind + shadcn/ui                │
└───────────────────────────┬──────────────────────────────────────────┘
                            │ HTTP / SSE
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     FastAPI Backend (Python 3.12)                     │
│                                                                      │
│  ┌─────────┐  ┌──────────┐  ┌──────────────────────────────────────┐│
│  │ /auth   │  │/research │  │      Rate Limiter (Redis)            ││
│  │register │  │ /upload  │  └──────────────────────────────────────┘│
│  │ login   │  │ /ingest  │                                          │
│  │ me      │  │ /query   │  ┌──────────────────────────────────────┐│
│  └─────────┘  │ /stream  │  │    JWT Auth (bcrypt + HS256)         ││
│               │/sessions │  └──────────────────────────────────────┘│
│               └────┬─────┘                                          │
│                    │                                                 │
│     ┌──────────────┴──────────────┐                                 │
│     │                             │                                  │
│     ▼                             ▼                                  │
│  /query (full agentic)      /query/stream (fast)                    │
│  ┌──────────────────┐       ┌──────────────────┐                    │
│  │   LangGraph      │       │  Direct Pipeline  │                   │
│  │                  │       │                   │                    │
│  │ Planner          │       │  Dense + Sparse   │                   │
│  │   ↓              │       │     ↓             │                   │
│  │ Retriever        │       │  RRF Fusion       │                   │
│  │  (HyDE+Graph)   │       │     ↓             │                   │
│  │   ↓              │       │  Cross-Encoder    │                   │
│  │ Grader           │       │     ↓             │                   │
│  │   ↓              │       │  Stream Generator │                   │
│  │ Contradiction    │       │  (SSE tokens)     │                   │
│  │   ↓              │       └──────────────────┘                    │
│  │ Generator        │                                               │
│  │   ↓              │                                               │
│  │ Citation Verif.  │                                               │
│  │   ↓              │                                               │
│  │ Hallucination    │                                               │
│  │ Checker          │                                               │
│  └──────────────────┘                                               │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │               RETRIEVAL LAYER                                  │  │
│  │                                                                │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │  │
│  │  │  Dense   │  │  Sparse  │  │  Graph   │  │  Web Search  │  │  │
│  │  │ (Qdrant) │  │ (PG FTS) │  │(Entities)│  │(Tavily/DDG)  │  │  │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────────────┘  │  │
│  │       └──────────────┼──────────────┘                          │  │
│  │                      ▼                                         │  │
│  │           Reciprocal Rank Fusion (RRF)                        │  │
│  │                      ↓                                         │  │
│  │           Cross-Encoder Reranker (ONNX)                       │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │               INGESTION PIPELINE                               │  │
│  │  Parse → Clean → Chunk → Filter → Embed → Store              │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │           BACKGROUND WORKERS (Celery + Redis)                  │  │
│  │  ingest_document_task  |  ingest_text_task                    │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
                            │                      │
              ┌─────────────┼──────────────────────┼───────────┐
              ▼             ▼                      ▼           ▼
        ┌──────────┐  ┌──────────┐          ┌──────────┐ ┌─────────┐
        │PostgreSQL│  │  Qdrant  │          │  Redis   │ │OpenRouter│
        │   16     │  │ (Vector) │          │ 7-Alpine │ │ (LLM)   │
        │          │  │          │          │          │ │Llama 3.3│
        │• Users   │  │• Vectors │          │• Queue   │ │  70B    │
        │• Docs    │  │• Payload │          │• Rate    │ │         │
        │• Chunks  │  │  (text,  │          │  Limits  │ │Gemma 3  │
        │• Convos  │  │  entities│          │• Results │ │  4B     │
        │• FTS idx │  │  domain) │          │          │ │         │
        └──────────┘  └──────────┘          └──────────┘ └─────────┘
```

---

## 1.3 Folder-by-Folder Analysis

### `backend/` — The Entire Server

| Folder | Purpose | Why It Exists | Key Interactions |
|--------|---------|---------------|------------------|
| `agents/` | LangGraph agent pipeline — nodes, state, graph | Orchestrates the full agentic RAG flow (planner → retriever → grader → generator → hallucination check) | Calls `retrieval/`, `prompts/`, `core/llm` |
| `api/` | FastAPI routes, request/response schemas, auth dependencies, rate limiting | Exposes HTTP endpoints for the frontend and external clients | Calls `agents/`, `ingestion/`, `memory/`, `workers/` |
| `core/` | Settings, LLM client singleton, security (JWT/bcrypt) | Centralized config — every module reads from here | Used by EVERYTHING |
| `db/` | SQLAlchemy models (User, Document, Chunk, Conversation), Postgres engine, Qdrant client | Data layer — defines what gets persisted and how | Used by `api/`, `ingestion/`, `retrieval/`, `memory/` |
| `ingestion/` | Document parsing, text cleaning, chunking, embedding, entity extraction, RAPTOR, multimodal | Transforms raw documents into searchable chunks with vectors | Calls `db/`, `core/`, stores to Qdrant + Postgres |
| `retrieval/` | Dense, sparse, hybrid, HyDE, graph expander, reranker, web search, context assembly | Finds the most relevant chunks for a query | Called by `agents/` and `api/routes/research.py` |
| `memory/` | Conversation history — in-memory cache + Postgres persistence | Enables multi-turn conversations with context | Called by `api/routes/research.py` |
| `workers/` | Celery app + background tasks | Async document ingestion so the API doesn't block | Called by `api/` and `agents/` (web search ingest) |
| `prompts/` | Prompt templates for the generator | Separates prompt engineering from business logic | Called by `agents/nodes.py` |
| `evaluation/` | LLM-as-judge evaluator + schemas | Measures pipeline quality (faithfulness, relevancy, recall, precision) | Called by `scripts/run_eval.py` |
| `observability/` | Placeholder for tracing (Langfuse integration is inline in `agents/nodes.py`) | Future: centralized tracing | Currently empty `__init__.py` |

### `frontend/` — Next.js Chat UI

| Folder | Purpose |
|--------|---------|
| `app/page.tsx` | Main chat interface — message input, streaming display, session management |
| `app/login/` | Login page |
| `app/signup/` | Registration page |
| `app/ingest/` | Document upload/ingestion page |
| `components/upload-modal.tsx` | In-chat file upload modal |
| `lib/api.ts` | API client — handles auth headers, SSE streaming, all backend calls |
| `lib/auth-context.tsx` | React context for JWT token management |
| `lib/auth.ts` | Token storage helpers (localStorage) |

### `scripts/` — Operational Tools

| File | Purpose |
|------|---------|
| `arxiv_ingest.py` | Search arXiv → download PDFs → ingest into Verity |
| `build_raptor.py` | Build RAPTOR cluster summaries over existing chunks |
| `seed_kb.py` | Seed the knowledge base with sample documents |
| `reset_kb.py` | Clear the knowledge base |
| `run_eval.py` | Run the LLM-as-judge evaluation (baseline vs reranker) |

### `migrations/` — Database Migrations

Alembic migrations for PostgreSQL schema changes. `env.py` connects to the async engine.

### `infrastructure/` — Docker for Infrastructure

`docker-compose.yml` for Postgres + Redis only (lightweight alternative to the full compose).

---

## 1.4 File-by-File Analysis (Critical Files)

### [main.py](file:///Users/abhijaat/Desktop/Verity/Verity/main.py) — Application Entry Point

**Purpose**: Creates the FastAPI app, configures CORS, registers routers, initializes Qdrant collection on startup.

**Execution Flow**:
1. Configure Python logging (structured format for `verity.*` loggers)
2. Define `lifespan` context manager → calls `init_collection()` on startup
3. Create `FastAPI(title="Verity")` instance
4. Add CORS middleware (origins from `CORS_ORIGINS` env var)
5. Include `auth_router` and `research_router`
6. Register `/health` endpoint

**Key Decision**: `lifespan` (not `@app.on_event`) because FastAPI deprecated startup/shutdown events in favor of ASGI lifespan.

---

### [backend/core/config.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/core/config.py) — Settings

**Purpose**: Single source of truth for ALL configuration. Uses `pydantic-settings` to load from `.env`.

**Critical Settings**:
- `openrouter_api_key` — LLM access
- `llm_model` — Primary model (`meta-llama/llama-3.3-70b-instruct:free`)
- `fast_llm_model` — Lightweight model for domain checks, HyDE (`google/gemma-3-4b-it:free`)
- `embedding_model` / `embedding_dimension` — `BAAI/bge-small-en-v1.5` / 384
- `enable_reranker` — Cross-encoder reranking (default: ON)
- `web_fallback_threshold` — If best dense cosine < this, fall back to web search (default: 0.6)
- `enable_entity_extraction` — Per-chunk LLM entity extraction (default: OFF — too slow)

**Why pydantic-settings**: Type-safe, auto-validates, auto-loads `.env`, provides defaults, supports `Field(description=...)` for documentation.

---

### [backend/core/llm.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/core/llm.py) — LLM Client Singleton

**Purpose**: Single OpenAI-compatible client pointing at OpenRouter. Every module imports `llm` and `provider_kwargs()` from here.

**Key Design**: Uses `openai.OpenAI` with `base_url="https://openrouter.ai/api/v1"` — OpenRouter provides an OpenAI-compatible API that routes to 100+ models. `provider_kwargs()` optionally pins a specific provider (e.g., Fireworks) to avoid caching issues.

---

### [backend/agents/graph.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/agents/graph.py) — LangGraph Definition

**Purpose**: Defines the full agentic RAG pipeline as a `StateGraph`.

**Flow**:
```
START → planner → retriever → grader
                                ├─ relevant → contradiction_detector → generator → citation_verifier → hallucination_checker → END
                                └─ not relevant → web_search → generator → citation_verifier → hallucination_checker → END
```

**Conditional Edges**:
- `_route_after_grader`: If chunks are relevant → contradiction detection. Otherwise → web search.
- `_route_after_web_search`: If web found results → generator. Otherwise → END (no answer).

---

### [backend/agents/state.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/agents/state.py) — Agent State

**Purpose**: TypedDict defining ALL data flowing through the LangGraph.

**Fields** (22 total):
- `query`, `domain`, `user_id` — Input context
- `db` — Database session (passed through, never serialized)
- `session_id`, `conversation_history` — Memory
- `plan`, `sub_queries` — Planner output
- `chunks`, `context`, `answer` — Retrieval + generation
- `contradictions` — Detected conflicts between sources
- `is_relevant`, `has_hallucination`, `citation_verified` — Quality signals
- `web_search_used`, `web_sources`, `sources` — Source tracking

---

### [backend/agents/nodes.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/agents/nodes.py) — Agent Nodes (478 lines)

**Purpose**: Implements every node function in the LangGraph.

| Node | What It Does | LLM Call? |
|------|-------------|-----------|
| `planner_node` | Analyzes query complexity, decomposes complex queries into 2-3 sub-queries | Yes (main model) |
| `retriever_node` | Runs hybrid retrieval (or per-sub-query retrieval + merge) | No (embedding only) |
| `grader_node` | LLM judges if chunks are relevant to the query | Yes |
| `contradiction_detector_node` | LLM checks if chunks contradict each other | Yes |
| `web_search_node` | Falls back to web search, converts results to fake chunks, triggers background KB ingest | No (HTTP) |
| `generator_node` | Assembles context, generates answer with citations | Yes (+ Langfuse trace) |
| `stream_generator_node` | Streaming version — yields tokens as SSE | Yes (stream=True) |
| `citation_verifier_node` | Parses `[1][2]` citations, asks LLM if each is supported by the chunk | Yes |
| `hallucination_checker_node` | LLM checks if the entire answer is grounded in context | Yes |

---

### [backend/ingestion/pipeline.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/pipeline.py) — Ingestion Pipeline (7 Steps)

**The most important file for understanding RAG ingestion.**

**Step-by-step**:
1. **Detect type** — file extension or URL → `pdf`, `docx`, `web`, `txt`, `md`
2. **Hash deduplication** — SHA-256 of file; if same user + same hash exists → skip
3. **Parse** — Convert document to `list[ParsedPage]` (text + page number)
4. **Clean** — Remove emails, DOIs, copyright, page numbers, references sections
5. **Chunk** — Split into `TextChunk` objects using type-appropriate chunker
6. **Filter** — Drop chunks with < 30 words, low alpha ratio, reference entries
7. **Embed** — Batch embed all chunks using `fastembed` (BAAI/bge-small-en-v1.5, 384-dim)
8. **Store** — Upsert vectors + payload to Qdrant, bulk insert chunks to Postgres

---

## 1.5 Request Lifecycle

### Streaming Query (`POST /research/query/stream`) — What the Chat UI Uses

```
User types question in chat
        │
        ▼
Browser sends POST with {query, domain, session_id, top_k}
+ Authorization: Bearer <JWT>
        │
        ▼
FastAPI: get_current_user dependency
  → decode JWT → load User from Postgres
        │
        ▼
Rate limiter: check_rate_limit(user_id)
  → Redis INCR on key "rl:{user_id}"
  → 20 requests per 20 minutes
        │
        ▼
Load conversation history from Postgres (last 5 Q&A pairs)
        │
        ▼
Domain guard: ask fast LLM "is this query about {domain}?"
  → If off-domain: stream a polite refusal
        │
        ▼
HybridRetriever.retrieve() [fast path: HyDE=OFF, graph=OFF]
  ├─ Dense: embed query → Qdrant cosine search (top 20)
  ├─ Sparse: PostgreSQL FTS with tsvector/GIN (top 20)
  └─ RRF fusion → Cross-encoder rerank (top 30 → top_k)
        │
        ▼
Check best_dense_score >= web_fallback_threshold (0.6)?
  ├─ YES: Use KB chunks as context
  └─ NO:  Web search (Tavily → DuckDuckGo fallback)
          → Use web results as context
          → Fire-and-forget: Celery ingest web results into KB
        │
        ▼
stream_generator_node(): Build prompt with context + history
  → OpenRouter LLM call with stream=True
  → Yield each token as SSE: data: {"type":"token","content":"..."}
        │
        ▼
After stream completes:
  → Emit sources event with chunk details
  → Emit pipeline transparency event
  → Save exchange to conversation history (Postgres)
  → Emit done event
```

### Full Agentic Query (`POST /research/query`) — Complete Pipeline

```
Same auth + rate limit + history loading
        │
        ▼
graph.ainvoke(initial_state) — runs the LangGraph:
        │
        ▼
1. PLANNER: Analyze query complexity
   → simple/moderate/complex
   → Generate 0-3 sub-queries
        │
        ▼
2. RETRIEVER: For each sub-query (or main query):
   → HyDE (generate hypothetical answer, embed THAT)
   → Dense search (Qdrant, top 20)
   → Sparse search (Postgres FTS, top 20)
   → Graph entity search (Qdrant, top 10)
   → RRF fusion → Cross-encoder rerank → top 5
        │
        ▼
3. GRADER: "Are these chunks relevant to the query?"
   → LLM returns {"relevant": true/false}
   ├─ relevant → CONTRADICTION DETECTOR
   └─ not relevant → WEB SEARCH
        │                    │
        ▼                    ▼
4a. CONTRADICTION        4b. WEB SEARCH
    DETECTOR                 Tavily/DDG
    LLM checks if            → Convert to fake chunks
    chunks conflict           → Queue KB ingest
        │                    │
        └────────┬───────────┘
                 ▼
5. GENERATOR: Assemble context + prompt
   → LLM generates answer with [1][2] citations
   → Log to Langfuse (if configured)
        │
        ▼
6. CITATION VERIFIER: Parse [1][2] from answer
   → LLM checks each cited claim against the chunk
   → If unverified: append warning to answer
        │
        ▼
7. HALLUCINATION CHECKER: "Is the answer grounded in context?"
   → LLM returns {"grounded": true/false}
   → If hallucinated: append warning
        │
        ▼
Return QueryResponse {answer, chunks_used, sources[]}
```

---

# PHASE 2 — COMPLETE RAG MASTERY

---

## 2.1 RAG Fundamentals

### What is RAG?

**Definition**: Retrieval-Augmented Generation — a technique that combines information retrieval with LLM generation to produce grounded, verifiable answers.

**Why It Exists**: LLMs suffer from three fundamental limitations:
1. **Hallucinations** — They generate plausible-sounding but false information
2. **Knowledge cutoffs** — Training data has a date boundary; they don't know recent facts
3. **Context windows** — Even with 128K context, you can't fit an entire knowledge base

**The Core Insight**: Instead of asking the LLM to remember everything, *retrieve* the relevant information and give it as context. The LLM becomes a reasoning engine over retrieved evidence, not a memory bank.

**How It Works (3 Steps)**:
```
1. RETRIEVE: Query → Embedding → Vector Search → Top-K Chunks
2. AUGMENT:  Chunks formatted as context in the prompt
3. GENERATE: LLM reads context + question → produces grounded answer
```

**Mathematical Intuition**:
```
P(answer | question) ← Traditional LLM (paramtric memory only)

P(answer | question, retrieved_docs) ← RAG (grounded in evidence)
```
By conditioning on retrieved documents, the model is constrained to information that actually exists.

**Verity Implementation**:
- Step 1: [hybrid_retriever.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/retrieval/hybrid_retriever.py) — Dense + Sparse + Graph + RRF + Rerank
- Step 2: [context_assembler.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/retrieval/context_assembler.py) — Format chunks as numbered context
- Step 3: [nodes.py generator_node](file:///Users/abhijaat/Desktop/Verity/Verity/backend/agents/nodes.py#L202-L247) — LLM generates answer

**Interview Explanation**:
> "RAG solves the fundamental problem of LLM hallucination by decoupling knowledge storage from reasoning. Instead of baking knowledge into model weights through training, we store it in an external searchable database and retrieve only the relevant pieces at query time. This gives us updateability without retraining, verifiability through source citations, and factual grounding that pure parametric models lack."

### Retrieval vs Fine-tuning

| Dimension | RAG | Fine-tuning |
|-----------|-----|-------------|
| **Knowledge update** | Instant (update docs) | Expensive (retrain) |
| **Cost** | Retrieval compute | GPU training cost |
| **Verifiability** | Citations possible | No attribution |
| **Best for** | Facts, recent info | Style, format, behavior |
| **Hallucination** | Reduced (grounded) | Can increase |
| **Verity's choice** | ✅ RAG | Not used |

---

## 2.2 Document Ingestion

### The Ingestion Pipeline

**What**: The process of converting raw documents into searchable, embedded chunks.

**Why**: LLMs can't process entire documents. We need to break documents into semantically meaningful pieces, convert them to vectors, and store them for fast retrieval.

**Verity's 7-Step Pipeline** (from [pipeline.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/pipeline.py)):

```
Document (PDF/DOCX/URL/TXT)
    ↓
Step 1: DETECT TYPE → file extension / URL detection
    ↓
Step 2: HASH DEDUP → SHA-256, skip if same user+hash exists
    ↓
Step 3: PARSE → Extract text from format (PyMuPDF/python-docx/trafilatura)
    ↓
Step 4: CLEAN → Remove emails, DOIs, copyright, references, page numbers
    ↓
Step 5: CHUNK → Split into 800-char chunks with 150-char overlap
    ↓
Step 6: FILTER → Drop chunks < 30 words, low alpha ratio, reference entries
    ↓
Step 7: EMBED + STORE → bge-small-en-v1.5 (384-dim) → Qdrant + Postgres
```

**Production Decisions in Verity**:
- **Hash dedup is scoped per user** — two users can upload the same file (UniqueConstraint on `user_id` + `file_hash`)
- **Cleaning happens BEFORE chunking** — if you chunk first, noise ends up in chunks
- **Filtering happens AFTER chunking** — drop low-quality chunks post-split
- **Dual storage** — Qdrant (vectors for dense search) + Postgres (text for sparse FTS search)

---

## 2.3 Chunking

### Why Chunking Exists

**The Problem**: Embedding models have token limits (typically 512 tokens). Even if they accepted more, long texts produce diluted embeddings where the semantic signal is averaged across too many topics. A single embedding for a 20-page paper would match every query vaguely but none specifically.

**The Solution**: Split documents into small, focused chunks (300-1000 chars). Each chunk has one coherent topic → produces a focused embedding → matches relevant queries precisely.

### Verity's Chunking Strategies

**1. RecursiveChunker** — Used for PDF, TXT (default)

```python
# From backend/ingestion/chunking/recursive_chunker.py
self.splitter = RecursiveCharacterTextSplitter(
    chunk_size=800,        # max characters per chunk
    chunk_overlap=150,     # overlap preserves context across boundaries
    separators=["\n\n", "\n", ".", " ", ""],  # try paragraph → line → sentence → word
)
```

**Why recursive**: It tries the highest-level separator first (`\n\n` = paragraph). If a paragraph is still too long, it falls back to `\n` (line), then `.` (sentence), then ` ` (word). This preserves semantic coherence.

**Why 800 chars / 150 overlap**: 800 chars ≈ 150-200 words ≈ ~200 tokens. Well within the 512-token embedding model limit. 150-char overlap ensures context isn't lost at chunk boundaries.

**2. HeadingChunker** — Used for DOCX

```python
# DOCXParser already splits by headings → each ParsedPage is one section
# HeadingChunker keeps sections that fit within max_chunk_size as-is
# Only splits further with RecursiveCharacterTextSplitter if a section exceeds 800 chars
```

**Why**: DOCX files have explicit heading structure. Splitting by heading preserves the author's logical organization.

**3. MarkdownChunker** — Used for Markdown files

```python
# Uses LangChain's MarkdownTextSplitter
# Respects markdown heading structure (# ## ###) and code blocks
```

**4. SemanticChunker** — Used for web content

```python
# CURRENTLY: Delegates to RecursiveChunker
# PLANNED: Use embedding similarity to detect topic shifts
```

**Why the fallback**: True semantic chunking requires computing embeddings for every sentence and finding breakpoints where cosine similarity drops. `fastembed` doesn't support the streaming sentence-level comparison needed. The README is honest about this: "The semantic chunker currently falls back to recursive splitting."

### Chunk Size Selection Tradeoffs

| Smaller Chunks (200-400 chars) | Larger Chunks (800-1500 chars) |
|------|------|
| More precise retrieval | More context per chunk |
| Higher recall (more matches) | Fewer chunks to retrieve |
| Risk: too fragmented, missing context | Risk: diluted embeddings |
| Better for factoid QA | Better for synthesis/summary |

**Verity's choice (800)**: Balanced — enough context for research paper sections, not so large that embeddings become unfocused.

---

## 2.4 Embeddings

### What Are Embeddings?

**Definition**: A function `f(text) → vector ∈ ℝⁿ` that maps text into a dense numeric vector where semantically similar texts have similar vectors.

**Mathematical Intuition**:
```
"RAG reduces hallucinations"  → [0.23, -0.15, 0.87, ..., 0.42]  (384 dimensions)
"Retrieval helps with accuracy" → [0.21, -0.13, 0.85, ..., 0.40]  (close in vector space)
"The weather is sunny today"    → [-0.52, 0.71, -0.03, ..., 0.15] (far in vector space)
```

### Similarity Metrics

**Cosine Similarity** (used by Verity/Qdrant):
```
cos(A, B) = (A · B) / (||A|| × ||B||)
```
- Range: [-1, 1] (for normalized vectors: [0, 1])
- 1.0 = identical direction (most similar)
- 0.0 = orthogonal (unrelated)
- Verity uses cosine distance in Qdrant: `Distance.COSINE`

**Why cosine over dot product or Euclidean**:
- Cosine is magnitude-invariant — it only cares about direction
- A long document and a short query can still have cosine similarity 1.0
- Euclidean would penalize shorter texts

### Verity's Embedding Implementation

```python
# From backend/ingestion/embedding/embedding_service.py
from fastembed import TextEmbedding

class EmbeddingService:
    def __init__(self):
        self.model = TextEmbedding(model_name="BAAI/bge-small-en-v1.5")

    def embed(self, texts: list[str]) -> list[list[float]]:
        vectors = list(self.model.embed(texts))
        return [v.tolist() for v in vectors]
```

**Why `BAAI/bge-small-en-v1.5`**:
- 384 dimensions (compact, fast)
- ONNX runtime via `fastembed` (no PyTorch needed → ~3GB memory savings)
- No API key required (runs locally)
- Ranked well on MTEB benchmark for its size

**Why `fastembed` over `sentence-transformers`**:
- `sentence-transformers` requires PyTorch (~2GB)
- `fastembed` uses ONNX Runtime (~200MB)
- Same model, 10x smaller runtime footprint
- Production benefit: smaller Docker images, faster cold starts

---

## 2.5 Retrieval

### Dense Retrieval (Verity: [dense_retriever.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/retrieval/dense_retriever.py))

**How it works**:
1. Embed the query → get 384-dim vector
2. Search Qdrant for nearest neighbors by cosine similarity
3. Filter by `user_id` and `domain` using Qdrant payload filters
4. Return top-K chunks with scores

**Strengths**: Captures semantic meaning ("reduce hallucinations" matches "grounding generation")
**Weakness**: Misses exact keywords ("RAPTOR" ≠ "hierarchical summarization" in vector space)

### Sparse Retrieval (Verity: [sparse_retriever.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/retrieval/sparse_retriever.py))

**How it works**: PostgreSQL full-text search using `tsvector` + GIN index.

```sql
SELECT c.text, ts_rank(c.search_vector, plainto_tsquery('english', :query)) AS score
FROM chunks c
JOIN documents d ON c.document_id = d.id
WHERE c.search_vector @@ plainto_tsquery('english', :query)
ORDER BY score DESC
LIMIT :top_k
```

**BM25 Intuition** (PostgreSQL's `ts_rank` is BM25-like):
```
score(q, d) = Σ IDF(term) × (TF(term,d) × (k₁+1)) / (TF(term,d) + k₁ × (1-b+b×|d|/avgdl))
```
- **TF** (term frequency): How often the term appears in the document
- **IDF** (inverse document frequency): Rare terms get higher weight
- **Length normalization**: Long documents don't get unfair advantage

**Strengths**: Exact keyword matching, rare entity names, acronyms
**Weakness**: No semantic understanding ("reduce hallucinations" ≠ "grounding generation")

### Hybrid Retrieval (Verity: [hybrid_retriever.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/retrieval/hybrid_retriever.py))

**Why hybrid**: Dense and sparse retrieval have complementary strengths. Hybrid combines them.

**Reciprocal Rank Fusion (RRF)**:
```python
def _reciprocal_rank_fusion(results_lists, k=60):
    for results in results_lists:
        for rank, chunk in enumerate(results):
            key = f"{chunk.document_id}_{chunk.chunk_index}"
            scores[key] += 1.0 / (k + rank + 1)
```

**Mathematical Formula**:
```
RRF_score(d) = Σᵢ 1/(k + rankᵢ(d))
```
Where `k=60` is a dampening constant that prevents top-ranked results from dominating.

**Why RRF over simple score addition**: Different retrievers have different score scales (cosine: 0-1, BM25: 0-∞). You can't add them. RRF only uses ranks, which are always comparable.

### Reranking (Verity: [reranker.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/retrieval/reranker.py))

**Why reranking exists**: First-stage retrievers (bi-encoders) encode query and document independently. They're fast but miss the interaction between query and document tokens. A cross-encoder sees both together and captures fine-grained relevance.

**Verity's Implementation**:
```python
from fastembed.rerank.cross_encoder import TextCrossEncoder
_model = TextCrossEncoder(model_name="Xenova/ms-marco-MiniLM-L-6-v2")

def rerank(query, chunks, top_k):
    scores = list(_model.rerank(query, [c.text for c in chunks]))
    for chunk, score in zip(chunks, scores):
        chunk.score = float(score)
    return sorted(chunks, key=lambda c: c.score, reverse=True)[:top_k]
```

**Pipeline**: RRF produces ~30 fused candidates → Cross-encoder reranks → Take top 5-8

**Measured Impact** (from Verity's evaluation):
| Metric | Without Reranker | With Reranker | Improvement |
|--------|:---:|:---:|:---:|
| Context Recall | 0.32 | 0.45 | **+41%** |
| Context Precision | 0.63 | 0.73 | **+16%** |
| Faithfulness | 0.865 | 0.89 | +3% |

---

## 2.6 Advanced RAG Techniques

### HyDE — Hypothetical Document Embeddings (Implemented in Verity)

**Problem**: A short query like "What is RAG?" has a very different form from the answer text in a paper. The query embedding is far from the passage embedding even though they're about the same topic.

**Solution**: Ask the LLM to generate a hypothetical answer, then embed THAT instead of the query.

```python
# From backend/retrieval/hyde.py
_SYSTEM = "Write 2-3 sentences that directly answer this question as if from an academic paper."

def generate_hypothetical_document(query: str) -> str:
    response = _llm.chat.completions.create(
        model=settings.fast_llm_model,  # Gemma 3 4B (fast, cheap)
        messages=[{"role": "system", "content": _SYSTEM}, {"role": "user", "content": query}],
    )
    return response.choices[0].message.content.strip()
```

**Why it works**: The hypothetical answer is in the same "form" as real paper text → its embedding is closer to relevant passages in vector space.

**Tradeoff**: Adds one LLM call (~1-2s latency). That's why the streaming path disables it (`use_hyde=False`).

### Graph RAG (Entity-Based Retrieval) (Implemented in Verity)

**Problem**: Dense search misses chunks that share entities but use different language.

**Solution**: Extract key entities from each chunk at ingest time. At query time, extract entities from the query and find chunks with overlapping entities.

```python
# Ingestion: extract_entities(chunk.text) → ["rag", "retrieval augmented generation", "vector search"]
# Query: _extract_query_entities(query) → ["rag", "hallucination"]
# Match: Qdrant MatchAny filter on entities field
```

**Verity's design decision**: Entity extraction is OFF by default (`ENABLE_ENTITY_EXTRACTION=false`) because it requires one LLM call per chunk at ingest time. For 1,000 chunks, that's 1,000 LLM calls.

### RAPTOR — Recursive Abstractive Processing for Tree-Organized Retrieval (Implemented in Verity)

**Problem**: Broad questions ("What are the main approaches to RAG?") need information scattered across many chunks. Dense search returns the most relevant individual chunks, but misses the big picture.

**Solution**: Cluster all chunks using K-Means, summarize each cluster with an LLM, embed the summaries, and store them as "level-1 summary nodes" in the same vector store.

```python
# From backend/ingestion/raptor/summarizer.py
def build_raptor_summaries(domain=None):
    chunks = _fetch_base_chunks(domain)           # Scroll all base chunks from Qdrant
    n_clusters = max(2, min(len(chunks)//10, 50)) # Heuristic: 1 cluster per 10 chunks
    groups = _cluster(chunks, n_clusters)          # K-Means on vectors
    for group in groups:
        summary = _summarize_cluster(texts)        # LLM summarization
        vector = _embedder.embed_one(summary)      # Embed the summary
        # Store with chunk_type="summary", level=1
```

**At query time**: Both raw chunks AND summary nodes compete in the same Qdrant search. Broad queries match summaries; specific queries match raw chunks.

### Self-Improving RAG (Adaptive KB) (Implemented in Verity)

**Problem**: User asks about a topic not in the KB. Traditional RAG returns "I don't know."

**Verity's Solution**:
1. If `best_dense_score < web_fallback_threshold` → KB doesn't have good content
2. Search the web (Tavily or DuckDuckGo)
3. **Immediately**: Use raw web text as context for the current answer
4. **Background**: Fire-and-forget Celery task to ingest web results into KB
5. **Next query**: Same topic → direct KB hit, no web search needed

```python
# From backend/agents/nodes.py
def _trigger_background_ingest(results, domain, user_id):
    from backend.workers.tasks import ingest_text_task
    for r in results:
        ingest_text_task.delay(text=r["content"], title=r["title"], ...)
```

---

# PHASE 3 — PARSER DEEP DIVE

---

## 3.1 PDF Parser

### [backend/ingestion/parsers/pdf_parser.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/parsers/pdf_parser.py)

**Library**: PyMuPDF (`fitz`) — C-based, very fast, low memory.

**Theory**: PDFs are not text documents. They're page description programs — a series of commands ("draw character 'A' at position (72, 144)"). Text extraction must reconstruct reading order from character positions.

**How It Works**:
1. Open PDF with `fitz.open(source)`
2. For each page: `page.get_text()` extracts text in reading order
3. If `extract_images=True`: Extract images, send to vision LLM for descriptions
4. Append image descriptions as `[Figure: ...]` to page text
5. Return `list[ParsedPage]` (one per page)

**Multimodal Pipeline** ([image_describer.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/multimodal/image_describer.py)):
```python
def describe_image(image_bytes: bytes) -> str | None:
    if len(image_bytes) < 5_000:  # Skip tiny images (icons, dots)
        return None
    b64 = base64.standard_b64encode(image_bytes).decode("utf-8")
    # Uses Llama 4 Scout vision model via OpenRouter
    response = _llm.chat.completions.create(
        model="meta-llama/llama-4-scout-17b-16e-instruct",
        messages=[{"role": "user", "content": [
            {"type": "image_url", "image_url": {"url": f"data:image/png;base64,{b64}"}},
            {"type": "text", "text": "Describe this figure..."},
        ]}],
    )
```

**Production Considerations**:
- Memory: PyMuPDF is C-based, ~10x lower memory than Java-based alternatives (Apache Tika)
- Speed: ~100 pages/second for text extraction
- Tables: Basic text extraction (not structured table parsing)
- Images: Optional (OFF by default — each image = 1 vision LLM call)

**Common Problems**:
- Scanned PDFs → no text layer → need OCR (not implemented in Verity)
- Multi-column layouts → reading order can be wrong
- Watermarks → appear as text noise (Verity's cleaner handles some of this)

### Memory Version
```python
class PDFParser:
    def parse(self, source):
        with fitz.open(source) as doc:
            return [ParsedPage(text=page.get_text(), page_number=i+1)
                    for i, page in enumerate(doc) if page.get_text().strip()]
```

---

## 3.2 DOCX Parser

### [backend/ingestion/parsers/docx_parser.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/parsers/docx_parser.py)

**Library**: `python-docx` — reads Office Open XML format.

**Theory**: DOCX files are ZIP archives containing XML. Each paragraph has a style (Normal, Heading 1, Heading 2, etc.). The parser uses heading styles as natural section boundaries.

**How It Works**:
1. Open DOCX with `Document(source)`
2. Iterate paragraphs: if style starts with "Heading" → start new section
3. Accumulate body paragraphs under each heading
4. Return each heading+body as a separate `ParsedPage`
5. Fallback: if no headings found, treat entire document as one page

**Why heading-based splitting**: DOCX has explicit document structure. Using it produces more semantically coherent chunks than blind character splitting.

---

## 3.3 Web Parser

### [backend/ingestion/parsers/web_parser.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/parsers/web_parser.py)

**Library**: `trafilatura` — state-of-the-art web content extraction.

**How It Works**:
```python
downloaded = trafilatura.fetch_url(source)           # HTTP GET
text = trafilatura.extract(downloaded,
    include_comments=False,  # Skip user comments
    include_tables=True,     # Keep data tables
)
```

**Why trafilatura over BeautifulSoup**: trafilatura is purpose-built for *main content extraction*. It removes navigation, ads, sidebars, footers — things BeautifulSoup would include. It uses heuristics and ML to identify the article body.

---

## 3.4 Text Cleaner

### [backend/ingestion/cleaning/text_cleaner.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/cleaning/text_cleaner.py)

**Why cleaning matters**: Academic papers contain enormous amounts of noise — emails, DOIs, copyright notices, page numbers, reference lists. If these end up in chunks, they pollute embeddings and waste context window space.

**Page-level cleaning** (`clean_page_text`):
1. Strip control characters (NUL bytes from PDF extraction crash PostgreSQL)
2. Truncate at noise section headings (References, Bibliography, Acknowledgments)
3. Remove lines containing: emails, DOIs, arXiv IDs, copyright notices, ACM/IEEE headers, page numbers, ORCID URLs, bare URLs

**Chunk-level filtering** (`is_clean_chunk`):
1. Min 30 words (drops headers, footers, lone titles)
2. No more than 3 reference-style lines (`[1] Lewis et al...`)
3. No emails or DOIs
4. Alpha ratio > 0.45 (drops number tables, metadata)

---

# PHASE 4 — LANGCHAIN MASTERY

---

## 4.1 Why LangChain Exists

**Before LangChain**: Every LLM application required manual prompt construction, output parsing, chain-of-thought orchestration, retriever integration, and memory management. Each team built their own abstractions.

**What LangChain provides**: A framework of composable abstractions for LLM applications — chains, retrievers, loaders, splitters, memory, agents.

**Verity's relationship with LangChain**: **Minimal usage**. Verity uses only:
1. `langchain-text-splitters` — `RecursiveCharacterTextSplitter`, `MarkdownTextSplitter`
2. `langchain-core` — required by LangGraph (which is the actual orchestration framework)
3. `langchain-openai` — transitively via LangGraph

Verity deliberately **avoids** LangChain for:
- LLM calls (uses raw `openai.OpenAI` client)
- Retrievers (custom `DenseRetriever`, `SparseRetriever`, `HybridRetriever`)
- Memory (custom `conversation_store.py`)
- Embeddings (uses `fastembed` directly)

**Why this matters**: Understanding what LangChain does vs what you can build yourself is the difference between a course learner and a senior engineer.

---

## 4.2 LangChain's Text Splitters (Used in Verity)

### RecursiveCharacterTextSplitter

**What**: Splits text by trying separators in order of granularity.

**Internal Implementation** (simplified):
```python
class RecursiveCharacterTextSplitter:
    def __init__(self, chunk_size=800, chunk_overlap=150, separators=None):
        self.separators = separators or ["\n\n", "\n", ".", " ", ""]

    def split_text(self, text):
        # Try first separator
        for sep in self.separators:
            splits = text.split(sep)
            if all(len(s) <= self.chunk_size for s in splits):
                return self._merge_with_overlap(splits, sep)
        # If text still too long, try next separator recursively
```

**Verity's usage**:
```python
# From recursive_chunker.py
self.splitter = RecursiveCharacterTextSplitter(
    chunk_size=800,
    chunk_overlap=150,
    separators=["\n\n", "\n", ".", " ", ""],
)
```

### Pure Python equivalent (no LangChain):
```python
def split_recursive(text, chunk_size=800, overlap=150, separators=None):
    separators = separators or ["\n\n", "\n", ". ", " "]
    chunks = []
    for sep in separators:
        if len(text) <= chunk_size:
            chunks.append(text)
            break
        parts = text.split(sep)
        current = ""
        for part in parts:
            if len(current) + len(part) + len(sep) > chunk_size:
                if current:
                    chunks.append(current.strip())
                    # Overlap: keep the last `overlap` chars
                    current = current[-overlap:] + sep + part
                else:
                    current = part
            else:
                current = current + sep + part if current else part
        if current:
            chunks.append(current.strip())
        break
    return chunks
```

---

## 4.3 What Verity Builds Instead of LangChain

### Instead of LangChain Retrievers → Custom HybridRetriever

```python
# LangChain way:
from langchain.retrievers import EnsembleRetriever
retriever = EnsembleRetriever(retrievers=[dense, sparse], weights=[0.5, 0.5])

# Verity way: Full control over RRF, reranking, web fallback
class HybridRetriever:
    async def retrieve(self, query, db, top_k, domain, user_id):
        # Concurrent HyDE + sparse
        # Dense after HyDE
        # RRF fusion
        # Cross-encoder rerank
        # Return (chunks, best_dense_score)  ← score used for web fallback decision
```

**Why custom**: LangChain's `EnsembleRetriever` doesn't support:
- The RRF → cross-encoder rerank pipeline
- Returning the raw dense score alongside fused results
- Per-user filtering
- Async concurrent retrieval

### Instead of LangChain Memory → Custom conversation_store.py

```python
# LangChain way:
from langchain.memory import ConversationBufferWindowMemory
memory = ConversationBufferWindowMemory(k=5)

# Verity way: Two-tier memory (in-memory cache + Postgres persistence)
_cache: dict[str, deque] = defaultdict(lambda: deque(maxlen=10))  # Hot cache

async def load_history(session_id, db, user_id):  # Cold storage
    result = await db.execute(text("SELECT role, content FROM conversation_history ..."))
```

---

# PHASE 5 — LANGGRAPH COMPLETE MASTERY

---

## 5.1 Why LangGraph Exists

**Before LangGraph**: LangChain chains were sequential — A → B → C. You couldn't:
- Loop back (retry on failure)
- Branch conditionally (grade, then decide next step)
- Manage state across multiple steps
- Add persistence or human-in-the-loop

**What LangGraph provides**: A graph-based orchestration framework where:
- **Nodes** = functions that transform state
- **Edges** = control flow (sequential, conditional, cyclic)
- **State** = a TypedDict that flows through the graph

**The Core Insight**: Agent workflows are naturally graphs, not chains. A grader might send you back to retrieval. A hallucination checker might trigger re-generation. These require cycles, which chains can't express.

---

## 5.2 Core Concepts

### StateGraph

The container for the entire workflow. Parameterized by a state type.

```python
# From backend/agents/graph.py
workflow = StateGraph(AgentState)  # AgentState is a TypedDict
```

### State

A TypedDict that acts as the shared memory across all nodes. Every node reads from and writes to this state.

```python
# From backend/agents/state.py
class AgentState(TypedDict):
    query: str
    domain: str | None
    user_id: str | None
    db: Any                       # AsyncSession — per request
    session_id: str | None
    conversation_history: list
    plan: dict                    # Planner output
    sub_queries: list[str]        # Decomposed sub-questions
    chunks: list
    contradictions: list[dict]
    context: str
    answer: str
    is_relevant: bool
    has_hallucination: bool
    citation_verified: bool
    web_search_used: bool
    web_sources: list[dict]
    sources: list[dict]
```

**Key Design Decision**: The `db` field (AsyncSession) is passed through state but never serialized. This lets every node access the database without global state, but means the graph can't be checkpointed to disk (a deliberate tradeoff for simplicity).

### Nodes

Functions that take state, do work, return a partial state update.

```python
# Node signature: (state: AgentState) -> dict
def planner_node(state: AgentState) -> dict:
    query = state["query"]
    # ... LLM call ...
    return {"plan": plan, "sub_queries": sub_queries}
    # Only returned keys are updated in state
```

**Important**: Nodes return a **partial** dict. Only the keys present in the return value are updated. Other state keys remain unchanged.

### Edges

```python
# Sequential: A always goes to B
workflow.add_edge("planner", "retriever")

# Conditional: Router function decides next node
workflow.add_conditional_edges("grader", _route_after_grader)
# _route_after_grader returns "contradiction_detector" or "web_search"
```

### Conditional Edges (Verity's Routing Logic)

```python
def _route_after_grader(state: AgentState) -> str:
    """Relevant chunks → contradiction check. No chunks → web search."""
    if state["is_relevant"] and state["chunks"]:
        return "contradiction_detector"
    return "web_search"

def _route_after_web_search(state: AgentState) -> str:
    """Web results → generator. Nothing → END."""
    return "generator" if state["chunks"] else END
```

---

## 5.3 Verity's Graph — Line by Line

```python
def build_graph():
    workflow = StateGraph(AgentState)

    # Register all 8 nodes
    workflow.add_node("planner", planner_node)
    workflow.add_node("retriever", retriever_node)
    workflow.add_node("grader", grader_node)
    workflow.add_node("contradiction_detector", contradiction_detector_node)
    workflow.add_node("web_search", web_search_node)
    workflow.add_node("generator", generator_node)
    workflow.add_node("citation_verifier", citation_verifier_node)
    workflow.add_node("hallucination_checker", hallucination_checker_node)

    # Edges: define the flow
    workflow.add_edge(START, "planner")                          # Entry point
    workflow.add_edge("planner", "retriever")                    # Always retrieve after planning
    workflow.add_edge("retriever", "grader")                     # Always grade after retrieval
    workflow.add_conditional_edges("grader", _route_after_grader) # Branch: relevant or not?
    workflow.add_edge("contradiction_detector", "generator")      # Contradictions → generate
    workflow.add_conditional_edges("web_search", _route_after_web_search)  # Web results?
    workflow.add_edge("generator", "citation_verifier")          # Verify citations
    workflow.add_edge("citation_verifier", "hallucination_checker")  # Check hallucinations
    workflow.add_edge("hallucination_checker", END)              # Done

    return workflow.compile()
```

### Execution Flow Diagram

```
Input State:
{query: "What is RAG?", domain: "ml", user_id: "abc", db: <session>, ...defaults...}

┌──────────┐    ┌───────────┐    ┌────────┐
│ planner  │───→│ retriever │───→│ grader │
│          │    │           │    │        │
│ Updates: │    │ Updates:  │    │Updates:│
│ • plan   │    │ • chunks  │    │• is_   │
│ • sub_   │    │           │    │ relevant│
│  queries │    │           │    │        │
└──────────┘    └───────────┘    └───┬────┘
                                     │
                          ┌──────────┴──────────┐
                          │                     │
                    is_relevant=True      is_relevant=False
                          │                     │
                          ▼                     ▼
              ┌─────────────────┐     ┌────────────┐
              │ contradiction   │     │ web_search │
              │ detector        │     │            │
              │                 │     │ Updates:   │
              │ Updates:        │     │ • chunks   │
              │ • contradictions│     │ • web_*    │
              └────────┬────────┘     └──────┬─────┘
                       │                     │
                       └──────────┬──────────┘
                                  ▼
                          ┌──────────────┐
                          │  generator   │
                          │              │
                          │ Updates:     │
                          │ • answer     │
                          │ • context    │
                          │ • sources    │
                          └──────┬───────┘
                                 │
                          ┌──────▼───────────┐
                          │citation_verifier │
                          │                  │
                          │ Updates:         │
                          │ • citation_      │
                          │   verified       │
                          └──────┬───────────┘
                                 │
                          ┌──────▼───────────────┐
                          │hallucination_checker │
                          │                      │
                          │ Updates:             │
                          │ • has_hallucination  │
                          └──────┬───────────────┘
                                 │
                                END
```

### Graph Invocation

```python
# From backend/api/routes/research.py
graph = build_graph()
result = await graph.ainvoke({
    "query": request.query,
    "domain": request.domain,
    "user_id": user_id,
    "db": db,
    # ... all initial state values ...
})
```

**`ainvoke`**: Async execution. LangGraph runs each node, passes the state forward, follows edges, and returns the final state.

---

## 5.4 Node Implementation Deep Dive

### Planner Node — Intelligence Before Retrieval

```python
def planner_node(state: AgentState) -> dict:
    prompt = """You are a research query planner. Analyze this query and return a JSON plan.
    Query: {query}
    Return JSON: {"complexity": "simple"|"moderate"|"complex", "approach": "direct"|"multi_step"|"comparative", "sub_queries": [...]}"""

    response = _llm.chat.completions.create(model=settings.llm_model, ...)
    plan = json.loads(response.choices[0].message.content)
    return {"plan": plan, "sub_queries": plan.get("sub_queries", [])[:3]}
```

**Why a planner**: A simple query ("What is RAG?") doesn't need sub-query decomposition. A complex query ("Compare HyDE, RAPTOR, and GraphRAG") benefits from being split into 3 focused sub-queries, each retrieving its own chunks.

### Retriever Node — Sub-Query Aware

```python
async def retriever_node(state: AgentState) -> dict:
    sub_queries = state.get("sub_queries") or []

    if sub_queries:
        # Retrieve for each sub-query separately
        for sq in sub_queries:
            sq_chunks, _ = await _retriever.retrieve(sq, ...)
            # Merge, deduplicate by (document_id, chunk_index)
        return {"chunks": merged[:5]}

    # Simple query: single retrieval
    chunks, best_dense_score = await _retriever.retrieve(state["query"], ...)
    return {"chunks": chunks}
```

**Key insight**: Sub-query retrieval produces a *union* of relevant chunks across topics, then deduplicates and sorts by score. This gives better coverage for complex questions.

---

## 5.5 Streaming vs Full: Two Execution Paths

Verity has **two distinct execution paths** — a critical architectural decision:

| Feature | `/query` (Full Agentic) | `/query/stream` (Fast) |
|---------|:-:|:-:|
| Orchestrator | LangGraph | Direct Python |
| HyDE | ✅ | ❌ |
| Graph Retrieval | ✅ | ❌ |
| Planner | ✅ | ❌ |
| Grader | ✅ | ❌ |
| Contradiction Detection | ✅ | ❌ |
| Citation Verification | ✅ | ❌ |
| Hallucination Check | ✅ | ❌ |
| Streaming | ❌ (returns full response) | ✅ (SSE) |
| Latency | ~15-30 seconds | ~5-10 seconds |
| LLM Calls | 5-8 | 1 |
| Web Fallback | Via grader | Via dense score threshold |

**Why two paths**: The full agentic pipeline makes 5-8 LLM calls (planner + grader + contradiction + generator + citation + hallucination). That's 15-30 seconds on free-tier models. The chat UI needs real-time response, so the streaming path skips all quality checks and streams tokens as they arrive.

**Interview explanation**: "We made a deliberate latency vs quality tradeoff. The full agentic path runs 8 quality-assurance steps but takes 20+ seconds. The streaming path runs only retrieval + generation for 5-second response times. Both paths share the same retrieval and generation code — the streaming path just skips the LLM-based quality gates."
