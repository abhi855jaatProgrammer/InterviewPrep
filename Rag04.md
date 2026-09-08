# VERITY — Supplementary Deep Dives & Quick Reference

> Companion to the main learning roadmap.
> Phases 1-5 → [Part 1](file:///Users/abhijaat/.gemini/antigravity-ide/brain/920c1b6a-2c91-4897-b3e4-59568cd39138/verity_learning_roadmap_phase_01_to_05.md) |
> Phases 6-12 → [Part 2](file:///Users/abhijaat/.gemini/antigravity-ide/brain/920c1b6a-2c91-4897-b3e4-59568cd39138/verity_learning_roadmap_phase_06_to_12.md) |
> Phases 13-25 → [Part 3](file:///Users/abhijaat/.gemini/antigravity-ide/brain/920c1b6a-2c91-4897-b3e4-59568cd39138/verity_learning_roadmap_phase_13_to_25.md)

---

# A — COMPLETE FRONTEND ARCHITECTURE

---

## A.1 Technology Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Framework | Next.js | 16 (App Router) |
| Language | TypeScript | Strict |
| Styling | Tailwind CSS 4 + CSS Variables | Dark-only theme |
| Components | shadcn/ui (minimal usage) | Custom inline styles |
| Markdown | react-markdown + remark-gfm | GitHub-flavored |
| Build | Turbopack (dev) / Next.js (prod) | - |

## A.2 Page Architecture

```
frontend/app/
├── layout.tsx         ← Root layout (AuthProvider wraps all pages)
├── page.tsx           ← Chat page (655 lines — the main UI)
├── globals.css        ← Design tokens (CSS variables, dark theme)
├── error.tsx          ← Error boundary
├── login/page.tsx     ← Login form
├── signup/page.tsx    ← Registration form
└── ingest/page.tsx    ← Document upload/URL ingestion page
```

## A.3 Auth Flow (Frontend)

```
┌──────────────┐     POST /auth/login     ┌──────────────┐
│  Login Page  │ ─────────────────────────►│    FastAPI    │
│              │◄────────────────────────  │              │
│              │   {access_token: "..."}   │              │
└──────┬───────┘                           └──────────────┘
       │
       │ setToken(token) → localStorage
       │ getMe() → {id, email}
       │ setUser(user)
       │
       ▼
┌──────────────┐
│  AuthContext  │  ← React Context wraps entire app
│              │
│  user: {id,  │  ← Available via useAuth() hook
│   email}     │
│  login()     │
│  register()  │
│  logout()    │
└──────┬───────┘
       │
       │ Every API call: Authorization: Bearer <token>
       │ 401 response → clearToken() → redirect /login
       │
       ▼
┌──────────────┐
│   Chat Page  │  ← Protected: redirects to /login if no user
└──────────────┘
```

**Key implementation details** (from [auth-context.tsx](file:///Users/abhijaat/Desktop/Verity/Verity/frontend/lib/auth-context.tsx)):
- On mount: check localStorage for token → call `GET /auth/me` to validate
- If token is invalid/expired: clear it, show login
- `useAuth()` hook throws if used outside `AuthProvider` — catches misuse at dev time

## A.4 SSE Streaming — Client Side

The most complex frontend code. From [api.ts](file:///Users/abhijaat/Desktop/Verity/Verity/frontend/lib/api.ts#L108-L171):

```typescript
export async function queryKBStream(req, onEvent) {
  // 1. Abort controller — 90-second idle timeout
  const controller = new AbortController()
  let idle = setTimeout(() => controller.abort(), 90_000)
  const resetIdle = () => { clearTimeout(idle); idle = setTimeout(...) }

  // 2. Fetch with streaming body
  const res = await fetch("/research/query/stream", {
    method: "POST",
    headers: authHeaders({"Content-Type": "application/json"}),
    body: JSON.stringify(req),
    signal: controller.signal,  // Can be aborted
  })

  // 3. Handle special HTTP status codes
  if (res.status === 401) → redirect to login
  if (res.status === 429) → throw RateLimitError(retry_after)

  // 4. Read the stream line by line
  const reader = res.body.getReader()
  const decoder = new TextDecoder()
  let buffer = ""

  while (true) {
    const { done, value } = await reader.read()
    if (done) break
    resetIdle()  // Server is alive — reset timeout

    buffer += decoder.decode(value, { stream: true })
    const lines = buffer.split("\n")
    buffer = lines.pop() ?? ""  // Keep partial last line in buffer

    for (const line of lines) {
      if (line.startsWith("data: ")) {
        const event = JSON.parse(line.slice(6))
        onEvent(event)  // Dispatch to UI
      }
    }
  }
}
```

**SSE Event Types** (from the server):
```
data: {"type": "token", "content": "RAG"}          ← Append to answer
data: {"type": "token", "content": " is a"}         ← Append to answer
data: {"type": "sources", "sources": [...]}          ← Show source cards
data: {"type": "pipeline", "methods": [...]}         ← Show pipeline badges
data: {"type": "error", "content": "Rate limited"}   ← Show error
data: {"type": "done"}                               ← Stream complete
```

**How the chat page processes events** (from [page.tsx](file:///Users/abhijaat/Desktop/Verity/Verity/frontend/app/page.tsx#L152-L222)):
```typescript
await queryKBStream(
  { query, domain, session_id },
  (event) => {
    if (event.type === "token") {
      // Append token to the last assistant message
      setMessages(prev => {
        const updated = [...prev]
        updated[updated.length - 1].content += event.content
        return updated
      })
    } else if (event.type === "sources") {
      // Attach sources to the last assistant message
      updated[updated.length - 1].sources = event.sources
    } else if (event.type === "pipeline") {
      // Attach pipeline info badges
      updated[updated.length - 1].pipeline = event
    }
  }
)
```

## A.5 Chat Page State Management

```typescript
// Core state — no external state library (React useState only)
const [messages, setMessages]     = useState<Message[]>([])      // Chat history
const [input, setInput]           = useState("")                  // Current input
const [domain, setDomain]         = useState("all")               // Selected domain filter
const [loading, setLoading]       = useState(false)               // Streaming in progress
const [sessionId, setSessionId]   = useState<string | null>(null) // Current chat session
const [sessions, setSessions]     = useState<SessionItem[]>([])   // Sidebar session list
const [showSources, setShowSources] = useState<Record<number, boolean>>({})  // Source panel toggle
const [expandedChunk, setExpandedChunk] = useState<string | null>(null)      // Expanded source
const [rateLimitSeconds, setRateLimitSeconds] = useState<number | null>(null)
const [uploadModalOpen, setUploadModalOpen] = useState(false)
```

**Design decision**: No Redux, Zustand, or Jotai. Chat state is local to the page because there's only one page. Session list is fetched from the API. This keeps the frontend simple.

## A.6 Design System — CSS Variables

From [globals.css](file:///Users/abhijaat/Desktop/Verity/Verity/frontend/app/globals.css):

```css
:root {
  --background: #0d1017;         /* Deep dark background */
  --foreground: #e2e4eb;         /* Light text */
  --card: #13161f;               /* Card/panel surface */
  --primary: #4f8ef7;            /* Blue accent (buttons, links, logo) */
  --secondary: #1e2230;          /* Input backgrounds */
  --muted-foreground: #7c8094;   /* Subtle text */
  --destructive: #e55757;        /* Error state */
  --border: rgba(255,255,255,0.08);  /* Subtle borders */
  --sidebar: #111420;            /* Sidebar background */
}
```

**Design philosophy**: Dark-only, minimal, professional. No light mode. The gradient `linear-gradient(135deg, #4f8ef7, #7c5df7)` is the brand identity used on the logo, send button, and user message bubbles.

## A.7 Upload Modal Pattern

From [upload-modal.tsx](file:///Users/abhijaat/Desktop/Verity/Verity/frontend/components/upload-modal.tsx):

**UX Flow**:
1. User clicks paperclip button in chat input bar
2. Modal opens with backdrop blur
3. User drags/drops or clicks to select PDF/DOCX/TXT/MD
4. File name auto-populates the title field
5. User selects domain
6. Click "Upload & Start Chat"
7. `uploadFile()` → POST multipart/form-data to `/research/upload`
8. On success: `onUploaded(title)` callback
9. Parent (`page.tsx`): creates new session, names it after the paper, seeds with intro message
10. User can immediately start asking questions about the uploaded paper

**Key detail**: The modal uses `e.stopPropagation()` on the inner div to prevent backdrop click from closing while clicking inside the modal.

---

# B — ENTITY EXTRACTION DEEP DIVE

---

## [backend/ingestion/graph/entity_extractor.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/graph/entity_extractor.py)

**Purpose**: Extract key technical entities from chunk text at ingestion time.

**Prompt**:
```
Extract key technical entities from the text.
Return JSON: {"entities": ["entity1", "entity2", ...]}
Rules:
- Specific concepts, methods, models, datasets, algorithms, or metrics only
- Lowercase, 1-4 words max
- Max 10 entities
- No generic words like "paper", "study", "result"
```

**How it's used** (from [pipeline.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/pipeline.py)):
```python
if settings.enable_entity_extraction:
    entities = extract_entities(chunk.text)
    # Stored in Qdrant payload: {"entities": ["rag", "dense retrieval", ...]}
```

**Why OFF by default**: For 1000 chunks, this means 1000 LLM calls. On free-tier models with rate limits, that could take hours. The feature is designed for paid API keys or local models.

## [backend/retrieval/graph_expander.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/retrieval/graph_expander.py)

**At query time**:
1. Extract entities from the query: `_extract_query_entities("How does RAPTOR work?")` → `["raptor", "hierarchical retrieval"]`
2. Search Qdrant with `MatchAny` filter on the `entities` payload field
3. Combined with user_id and domain filters
4. Returns chunks that share entities with the query

```python
must = [
    FieldCondition(key="entities", match=MatchAny(any=entities)),
    FieldCondition(key="user_id", match=MatchValue(value=user_id)),
]
results = qdrant_client.query_points(
    collection_name=settings.qdrant_collection,
    query=vector,                    # Still uses vector for ranking
    query_filter=Filter(must=must),  # But pre-filters by entities
    limit=top_k,
)
```

**Key insight**: This is a **hybrid** approach — it uses entity matching for *filtering* and vector similarity for *ranking*. It's not pure graph traversal (no edges/relationships), but it captures some of the benefits of knowledge graph retrieval.

---

# C — RAPTOR IMPLEMENTATION DEEP DIVE

---

## [backend/ingestion/raptor/summarizer.py](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/raptor/summarizer.py) (130 lines)

**Algorithm**:
```
1. Fetch all base chunks from Qdrant (chunk_type != "summary")
2. Filter by domain if specified
3. Calculate n_clusters = max(2, min(len(chunks)//10, 50))
   → 1 cluster per 10 chunks, minimum 2, maximum 50
4. K-Means clustering on 384-dim vectors
5. For each cluster:
   a. Take first 8 chunks (max), first 500 chars each
   b. LLM: "Synthesize key ideas into one concise paragraph (5-7 sentences)"
   c. Embed the summary
   d. Store in Qdrant with:
      - chunk_type: "summary"
      - level: 1
      - source_chunk_count: N
      - document_id: "raptor_summary"
6. Return number of summaries created
```

**Why K-Means**: Simple, deterministic, works well for balanced clusters. The vectors are already normalized (cosine space), so K-Means on L2 distance is equivalent to spherical K-Means on cosine.

**Why limit to 8 chunks × 500 chars per cluster**: LLM context windows. 8 × 500 = 4,000 chars ≈ 1,000 tokens. Summarization works best with focused input.

**At retrieval time**: Summary nodes compete with base chunks in the same Qdrant search. The system doesn't distinguish — a broad query naturally matches summaries better (they cover more topics), while a specific query matches base chunks.

---

# D — DOCKER & DEPLOYMENT ARCHITECTURE

---

## [docker-compose.yml](file:///Users/abhijaat/Desktop/Verity/Verity/docker-compose.yml) — 6 Services

```
┌─────────────────────────────────────────────────────┐
│                   Docker Network                     │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ postgres │  │  qdrant  │  │  redis   │          │
│  │    :5432 │  │   :6333  │  │   :6379  │          │
│  │          │  │          │  │          │          │
│  │ health:  │  │ no health│  │ health:  │          │
│  │ pg_isready  │          │  │ redis-cli│          │
│  └────┬─────┘  └──────────┘  └────┬─────┘          │
│       │                            │                 │
│       │ depends_on: healthy        │ depends_on:     │
│       │                            │ healthy         │
│       ▼                            ▼                 │
│  ┌──────────────────────────────────────┐           │
│  │            backend :8000             │           │
│  │                                      │           │
│  │ CMD: alembic upgrade head &&         │           │
│  │      uvicorn main:app               │           │
│  │        --host 0.0.0.0               │           │
│  │        --port 8000                   │           │
│  │        --workers 2                   │           │
│  └──────────────────────────────────────┘           │
│                                                      │
│  ┌──────────────────────────────────────┐           │
│  │         celery_worker                │           │
│  │                                      │           │
│  │ CMD: celery -A backend.workers.      │           │
│  │       celery_app worker              │           │
│  │       --loglevel=info -P solo        │           │
│  └──────────────────────────────────────┘           │
│                                                      │
│  ┌──────────────────────────────────────┐           │
│  │          frontend :3000              │           │
│  │                                      │           │
│  │ Next.js production build             │           │
│  └──────────────────────────────────────┘           │
└─────────────────────────────────────────────────────┘
```

**Startup order**:
1. `postgres` + `redis` start first (no dependencies)
2. Wait for health checks (pg_isready, redis-cli ping)
3. `qdrant` starts (no health check — condition: service_started)
4. `backend` starts: runs `alembic upgrade head` → then `uvicorn`
5. `celery_worker` starts: separate process, same codebase
6. `frontend` starts: independent, connects via `NEXT_PUBLIC_API_URL`

**Port mapping** (host → container):
- `5433:5432` (Postgres — offset to avoid conflict with local Postgres)
- `6333:6333` (Qdrant)
- `6380:6379` (Redis — offset to avoid conflict)
- `8000:8000` (Backend API)
- `3000:3000` (Frontend)

**Volume persistence**: `postgres_data` and `qdrant_data` are named volumes — data survives container restarts.

---

# E — COMPLETE API SURFACE

---

| Method | Endpoint | Auth | Purpose | Response |
|--------|----------|:----:|---------|----------|
| POST | `/auth/register` | ❌ | Create account | `{access_token, token_type}` |
| POST | `/auth/login` | ❌ | Sign in | `{access_token, token_type}` |
| GET | `/auth/me` | ✅ | Current user info | `{id, email}` |
| POST | `/research/upload` | ✅ | Upload file (multipart) | `{document_id, title, status}` |
| POST | `/research/ingest` | ✅ | Sync ingest (blocks) | `{document_id, title, status}` |
| POST | `/research/ingest/async` | ✅ | Async ingest (Celery) | `{task_id, status, message}` |
| GET | `/research/tasks/{id}` | ✅ | Check task status | `{task_id, status, result?, error?}` |
| POST | `/research/query` | ✅ | Full agentic query | `{answer, chunks_used, sources[]}` |
| POST | `/research/query/stream` | ✅ | Streaming query (SSE) | Event stream |
| GET | `/research/sessions` | ✅ | List chat sessions | `[{session_id, preview, last_at, message_count}]` |
| GET | `/research/sessions/{id}/messages` | ✅ | Session messages | `[{role, content}]` |
| GET | `/health` | ❌ | Health check | `{status: "ok"}` |

---

# F — END-TO-END DATA FLOW TRACE

---

## A PDF's Journey Through Verity

```
┌─ USER ──────────────────────────────────────────────────────────────┐
│ 1. Drags "attention_is_all_you_need.pdf" onto Upload Modal         │
│ 2. Sets title: "Attention Is All You Need", domain: "ml"           │
│ 3. Clicks "Upload & Start Chat"                                     │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │
                                 ▼
┌─ FRONTEND (api.ts) ────────────────────────────────────────────────┐
│ const form = new FormData()                                         │
│ form.append("file", selectedFile)                                   │
│ form.append("title", "Attention Is All You Need")                   │
│ form.append("domain", "ml")                                         │
│ POST /research/upload  [multipart/form-data]                        │
│ Authorization: Bearer <JWT>                                         │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │
                                 ▼
┌─ FASTAPI (routes/research.py) ──────────────────────────────────────┐
│ get_current_user(JWT) → user_id = "abc-123"                         │
│ Save file: data/uploads/attention_is_all_you_need.pdf                │
│ Call: ingest_document(source=path, title=..., db=..., user_id=...)  │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │
                                 ▼
┌─ INGESTION PIPELINE (pipeline.py) ──────────────────────────────────┐
│                                                                      │
│ Step 1: detect_type("...pdf") → "pdf"                                │
│                                                                      │
│ Step 2: SHA-256 hash → check (user_id, file_hash) in documents       │
│         Not found → proceed                                          │
│                                                                      │
│ Step 3: PDFParser.parse("attention.pdf")                             │
│         → fitz.open() → 15 pages → 15 ParsedPage objects             │
│         → Each has page text + page number                           │
│                                                                      │
│ Step 4: clean_page_text() for each page                              │
│         → Remove emails, DOIs, "Published in...", page numbers       │
│         → Truncate at "References" section heading                   │
│                                                                      │
│ Step 5: RecursiveChunker.chunk(pages)                                │
│         → 800 chars, 150 overlap, separators: ["\n\n", "\n", ".", " "]│
│         → Produces ~45 TextChunk objects                             │
│                                                                      │
│ Step 6: Filter: drop chunks < 30 words, reference-heavy, low alpha   │
│         → ~40 chunks survive                                         │
│                                                                      │
│ Step 7: EmbeddingService.embed([40 chunk texts])                     │
│         → BAAI/bge-small-en-v1.5 (ONNX, local)                      │
│         → 40 vectors × 384 dimensions                                │
│                                                                      │
│ Step 8a: INSERT INTO documents (...) → document_id = "doc-456"       │
│                                                                      │
│ Step 8b: For each chunk:                                             │
│          INSERT INTO chunks (document_id, text, chunk_index,          │
│            page_number, word_count, qdrant_id, search_vector)         │
│          → search_vector = to_tsvector('english', chunk.text)         │
│                                                                      │
│ Step 8c: qdrant.upsert(collection="verity_chunks", points=[          │
│            PointStruct(                                               │
│              id=uuid4(),                                              │
│              vector=[0.23, -0.15, 0.87, ...],  # 384-dim             │
│              payload={                                                │
│                "text": "The dominant sequence transduction...",        │
│                "document_id": "doc-456",                              │
│                "user_id": "abc-123",                                  │
│                "domain": "ml",                                        │
│                "chunk_index": 0,                                      │
│                "page_number": 1,                                      │
│                "entities": [],  # extraction OFF by default           │
│                "chunk_type": "base"                                   │
│              }                                                        │
│            )                                                          │
│          ])                                                           │
│                                                                      │
│ Return: Document(id="doc-456", title="Attention...", status="done")   │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │
                                 ▼
┌─ FRONTEND ──────────────────────────────────────────────────────────┐
│ onUploaded("Attention Is All You Need")                              │
│ → Create new session UUID                                            │
│ → Name it after the paper                                            │
│ → Seed with: "📄 Attention Is All You Need is indexed and ready."    │
│ → User sees a fresh chat titled after the paper                      │
└──────────────────────────────────────────────────────────────────────┘

                    ═══════════════════════════
                    User types: "Explain self-attention"
                    ═══════════════════════════

┌─ FRONTEND → POST /research/query/stream ─────────────────────────────┐
│ {query: "Explain self-attention", domain: "ml", session_id: "sess-7"}│
└────────────────────────────────┬─────────────────────────────────────┘
                                 │
                                 ▼
┌─ BACKEND (routes/research.py) ──────────────────────────────────────┐
│                                                                      │
│ 1. Auth: decode JWT → user_id = "abc-123"                            │
│ 2. Rate limit: INCR rl:abc-123 → count=1 ≤ 20 → allowed             │
│ 3. Load history: SELECT FROM conversation_history WHERE session_id.. │
│    → 0 messages (new session)                                        │
│ 4. Domain guard: "Is 'explain self-attention' about ML?" → yes       │
│                                                                      │
│ 5. HybridRetriever.retrieve(query, db, top_k=8, domain="ml",        │
│                              user_id="abc-123")                      │
│    [HyDE=OFF, Graph=OFF — fast path]                                 │
│                                                                      │
│    5a. Dense: embed("explain self-attention") → 384-dim vector       │
│        → Qdrant search with filters:                                 │
│          user_id="abc-123" AND domain="ml"                           │
│        → Top 20 results, best_dense_score = 0.78                     │
│                                                                      │
│    5b. Sparse: PostgreSQL FTS                                        │
│        → ts_rank(search_vector, plainto_tsquery('self-attention'))    │
│        → Top 20 results                                              │
│                                                                      │
│    5c. RRF fusion: merge dense + sparse by rank                      │
│        → Top 30 fused candidates                                     │
│                                                                      │
│    5d. Cross-encoder rerank (if enabled)                              │
│        → ms-marco-MiniLM rescores all 30                             │
│        → Return top 8                                                │
│                                                                      │
│ 6. best_dense_score = 0.78 ≥ 0.6 threshold → use KB (no web search) │
│                                                                      │
│ 7. Look up document titles for source cards                          │
│                                                                      │
│ 8. stream_generator_node():                                          │
│    context = "[1] The dominant sequence transduction models..."       │
│              "[2] Self-attention, sometimes called intra-attention.." │
│    prompt = build_report_prompt(context, query, history)              │
│    → OpenRouter API call with stream=True                            │
│    → Yield tokens one by one                                         │
│                                                                      │
│ 9. SSE events:                                                       │
│    data: {"type":"token","content":"Self"}                            │
│    data: {"type":"token","content":"-attention"}                      │
│    data: {"type":"token","content":" is"}                             │
│    ... (hundreds of tokens) ...                                      │
│    data: {"type":"sources","sources":[{doc_id,chunk_idx,score,...}]}  │
│    data: {"type":"pipeline","methods":["Dense","Sparse BM25",...]}    │
│    data: {"type":"done"}                                              │
│                                                                      │
│ 10. Save exchange to conversation_history:                           │
│     INSERT (session_id, user_id, role="user", content=query)          │
│     INSERT (session_id, user_id, role="assistant", content=answer)    │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │
                                 ▼
┌─ FRONTEND (page.tsx) ───────────────────────────────────────────────┐
│                                                                      │
│ Token events: append to messages[last].content → re-render           │
│   → User sees characters appearing one by one                        │
│   → ReactMarkdown renders bold, lists, code blocks, [1][2] cites     │
│                                                                      │
│ Sources event: attach to messages[last].sources                      │
│   → "8 chunks · 5 sources" button appears                           │
│   → Click → expandable source cards with title, score, page number   │
│   → Click source → raw chunk text in monospace                       │
│                                                                      │
│ Pipeline event: attach to messages[last].pipeline                    │
│   → Badges: [Dense (Qdrant)] [Sparse BM25] [Cross-Encoder Rerank]   │
│                                                                      │
│ Done event: setLoading(false)                                        │
│   → Refresh session list (sidebar updates)                           │
│                                                                      │
│ Auto-scroll: bottomRef.scrollIntoView({ behavior: "smooth" })       │
└──────────────────────────────────────────────────────────────────────┘
```

---

# G — 50 ADDITIONAL INTERVIEW QUESTIONS

---

## Beginner (1–10)

1. **Q: What happens when you embed a document?**
   > The embedding model reads the text and outputs a fixed-size vector (e.g., 384 dimensions) where each number captures some aspect of the text's meaning. Similar texts produce similar vectors.

2. **Q: Why can't you just put the entire document in the LLM prompt?**
   > Context windows are finite (128K tokens max). A corpus of 20 papers exceeds any context window. Even if it fit, the model would struggle to find relevant information in a sea of text. Retrieval narrows the context to the most relevant passages.

3. **Q: What is a chunk?**
   > A chunk is a piece of a document (typically 100-400 words) that represents one coherent idea. Documents are split into chunks so each can be independently embedded and retrieved.

4. **Q: Why use cosine similarity instead of Euclidean distance?**
   > Cosine similarity measures direction, not magnitude. A short chunk and a long passage about the same topic point in the same direction even if their vector magnitudes differ.

5. **Q: What is an inverted index?**
   > A mapping from each word to the list of documents/chunks containing it. PostgreSQL's GIN index on tsvector is an inverted index. It makes keyword lookups O(1) instead of scanning all documents.

6. **Q: What does "grounding" mean in the context of LLMs?**
   > Grounding means constraining the LLM's output to be based on specific evidence (retrieved documents) rather than its parametric memory. It reduces hallucinations.

7. **Q: What is the difference between synchronous and asynchronous ingestion?**
   > Synchronous: the API waits for ingestion to complete before responding. Asynchronous: the API queues the work, returns immediately with a task ID, and a background worker processes it.

8. **Q: What is a JWT?**
   > JSON Web Token — a self-contained, signed token that encodes user identity. The server creates it on login and the client sends it with every request. The server verifies the signature without querying a database.

9. **Q: Why does Verity store data in both Qdrant and PostgreSQL?**
   > Qdrant stores vectors for semantic search. PostgreSQL stores text for keyword search (BM25 via tsvector). Different retrieval methods need different data structures.

10. **Q: What are Server-Sent Events (SSE)?**
    > A protocol where the server sends events over a long-lived HTTP connection. The client receives events as they arrive — perfect for streaming LLM tokens in real-time.

## Intermediate (11–25)

11. **Q: Why does Verity use `fastembed` instead of OpenAI's embedding API?**
    > Local ONNX inference: no API key needed, no per-token cost, no rate limits, no network latency, no data leaving the server. The tradeoff is slightly lower quality than frontier models like `text-embedding-3-large`.

12. **Q: What happens if the cross-encoder reranker is disabled?**
    > The RRF-fused results are returned directly. In Verity's evaluation, this drops context recall by 41% and precision by 16%. The reranker is the single biggest quality improvement.

13. **Q: Explain the web_fallback_threshold decision.**
    > We compare the raw dense cosine similarity (0-1) against a threshold (default 0.6). Below it, the KB doesn't have relevant content, so we search the web. We use the dense score specifically because RRF scores are rank-based (~0.02) and meaningless for absolute relevance.

14. **Q: Why does Verity use `openai.OpenAI` instead of `langchain.ChatOpenAI`?**
    > Direct SDK control. LangChain's wrapper adds abstraction that complicates debugging, doesn't support all OpenRouter-specific features, and adds a dependency for no benefit when you're only making direct API calls.

15. **Q: How does the domain guard work?**
    > Before retrieval, a fast LLM (Gemma 3 4B) is asked: "Is this query about {domain}? Yes or no." If the answer is "no", the system returns a polite refusal instead of retrieving irrelevant results.

16. **Q: Why is the Celery worker started with `-P solo`?**
    > `-P solo` means single-threaded, no multiprocessing. The ingestion tasks are I/O-bound (database writes, API calls) not CPU-bound. Solo pool avoids the complexity of process management and works well with async code.

17. **Q: How does hash deduplication work and what's its limitation?**
    > SHA-256 hash of file contents, scoped per user (UniqueConstraint on user_id + file_hash). Limitation: URLs don't have file hashes — the same web page can be ingested multiple times.

18. **Q: What is the `_CONTROL_CHARS` regex for?**
    > PDF text extraction can produce NUL bytes (0x00) and other control characters. PostgreSQL rejects strings containing NUL. The regex strips all control characters before any database interaction.

19. **Q: Why does the generator prompt say "Do NOT add information from outside the provided context"?**
    > This is the grounding instruction. Without it, the LLM would mix retrieved facts with its parametric memory, making hallucinations indistinguishable from grounded claims.

20. **Q: How does the conversation memory work across sessions?**
    > Each session has a UUID. Messages are stored in `conversation_history` (Postgres) with the session_id. On each query, the last 5 Q&A pairs are loaded and included in the prompt for continuity. An in-memory deque caches the last 10 messages per session for fast access.

21. **Q: What is the difference between the planner deciding "complex" vs "simple"?**
    > Simple: single retrieval pass. Complex: the planner decomposes into 2-3 sub-queries, retrieves for each separately, deduplicates and merges the results. This gives better coverage for multi-faceted questions.

22. **Q: How does Verity handle LLM rate limits gracefully?**
    > Multiple levels: (1) User-facing rate limiter (Redis, 20 req/20 min). (2) Stream path catches 429 errors and sends a friendly SSE error event. (3) Free-tier LLM models have built-in rate limits — errors are caught and displayed to the user.

23. **Q: Why does the streaming path skip the hallucination checker?**
    > The hallucination checker requires a full answer to evaluate. In streaming, the answer is being generated token-by-token. Waiting for the full answer, then making another LLM call, would add ~6 seconds after the user already sees the response. The tradeoff is acceptable for chat.

24. **Q: What makes the evaluation framework "LLM-as-judge" rather than traditional metrics?**
    > Traditional metrics (BLEU, ROUGE) measure word overlap. They can't assess whether an answer is *faithful to its sources* or *actually answers the question*. The LLM judge reads the answer and context, then scores semantic quality on a 0-1 scale.

25. **Q: How does Verity's self-improving KB work?**
    > When a query triggers web search fallback: (1) Web results are used immediately as raw context for the current answer. (2) In the background, a Celery task ingests each web result through the full pipeline (clean, chunk, embed, store). (3) The next similar query finds the content in the KB directly — no web search needed.

## Senior/Staff (26–50)

26. **Q: Design the caching layer for Verity at 100K users.**
    > Semantic query cache: embed the query, search a cache collection for similar past queries (cosine > 0.95). If found, return the cached answer. Use Redis for TTL management. Invalidate when new documents are ingested for that user. Cache hit rate should be ~30-40% for research-heavy users.

27. **Q: The `tsvector` search uses `plainto_tsquery`. When would you switch to `websearch_to_tsquery`?**
    > `plainto_tsquery` treats all words as AND. `websearch_to_tsquery` supports OR, NOT, and quoted phrases. For user-facing search where users might type "RAG OR retrieval -fine-tuning", `websearch_to_tsquery` gives better control.

28. **Q: Why does Verity pass `db: AsyncSession` through the LangGraph state instead of using a global?**
    > Request-scoped sessions. Each API request gets its own database session from the connection pool. If nodes shared a global session, concurrent requests would corrupt each other's transactions. Passing through state ensures isolation.

29. **Q: What would you change about the entity extraction approach?**
    > Current: one LLM call per chunk (expensive, slow, OFF by default). Better: use a NER model (spaCy, GLiNER) that runs locally in milliseconds. Or: extract entities from the full document once, then propagate to chunks. Or: use the embedding model itself to identify entity-like substrings.

30. **Q: How would you implement streaming for the full agentic path?**
    > LangGraph supports streaming via `astream_events()`. Each node can yield intermediate results. The planner could emit "Planning..." The grader could emit "Evaluating relevance..." The generator streams tokens. This gives the user visibility into the pipeline while maintaining all quality checks.

31. **Q: The rate limiter uses a sliding window counter. What's wrong with this approach at scale?**
    > Redis INCR is atomic, but the window is fixed from the first request. A user can make 20 requests in the last second of one window and 20 in the first second of the next — 40 requests in 2 seconds. A true sliding window (Redis sorted set with timestamps) prevents this burst.

32. **Q: How would you add RBAC (Role-Based Access Control)?**
    > Add a `role` field to the User model (admin, researcher, viewer). Create a `require_role(role)` dependency. Admin: can manage all users' documents. Researcher: can ingest and query. Viewer: query only. Add RLS in Postgres for database-level enforcement.

33. **Q: What's the failure mode if Qdrant loses data?**
    > Dense retrieval fails completely. Sparse (Postgres) still works. The system degrades to keyword-only search. Recovery: re-embed all chunks from Postgres text (chunks table has full text). This is why dual-write is valuable — Postgres is the source of truth.

34. **Q: How would you implement document-level permissions?**
    > Add an `access_list` table: (document_id, user_id, permission). At retrieval time, join or pre-filter by accessible document IDs. In Qdrant, this requires either a list-type payload filter or a separate collection per access group.

35. **Q: Compare Verity's approach to RAG with LlamaIndex's approach.**
    > LlamaIndex provides pre-built index types (VectorStoreIndex, TreeIndex, KnowledgeGraphIndex) with built-in retrieval. Verity builds each component from scratch. LlamaIndex is faster to prototype; Verity gives full control over every decision (chunk size, fusion weights, rerank pipeline, web fallback logic).

36. **Q: The `pool_pre_ping=True` setting — what problem does it solve?**
    > Managed Postgres services (Neon) scale to zero after inactivity. When they wake up, existing connections in the pool are dead. `pool_pre_ping` sends a lightweight check before each query. If the connection is dead, it's replaced silently instead of raising an error.

37. **Q: Why `expire_on_commit=False` in the session maker?**
    > After `commit()`, SQLAlchemy expires all loaded attributes by default, meaning the next attribute access triggers a lazy load (another query). With `expire_on_commit=False`, committed objects remain usable without extra queries. Essential for async code where lazy loading is problematic.

38. **Q: How would you migrate from user_id-based filtering to PostgreSQL Row-Level Security?**
    > (1) Create a Postgres role per application context. (2) Enable RLS on all tables: `ALTER TABLE documents ENABLE ROW LEVEL SECURITY`. (3) Create policy: `CREATE POLICY user_docs ON documents USING (user_id = current_setting('app.user_id')::uuid)`. (4) Set `app.user_id` at the start of each request. (5) Remove WHERE clauses from application code — the database enforces isolation.

39. **Q: The citation verifier parses `[1][2]` from the answer. What's the edge case it doesn't handle?**
    > Numeric references in non-citation contexts: "The model achieved [1] percent accuracy" or "[3] years of training data". The current regex `r'\[(\d+)\]'` can't distinguish citations from other bracketed numbers. A more robust approach: require citations in a specific format like `[Source 1]` or use a separate citation token.

40. **Q: How would you implement A/B testing between retrieval strategies?**
    > (1) Feature flags in settings (e.g., `RETRIEVAL_STRATEGY=hybrid_v2`). (2) Traffic splitting: hash(user_id) % 100 → assign to control or treatment. (3) Log all retrieval scores, chunks, and user feedback per strategy. (4) Run the evaluation framework weekly per strategy. (5) Compare faithfulness, relevancy, recall, precision. (6) Promote the winner.

41. **Q: What's the latency breakdown of a streaming query?**
    > Embedding: ~50ms (local ONNX). Dense search: ~20ms (Qdrant). Sparse search: ~30ms (Postgres). RRF: ~1ms. Reranking: ~200ms (30 candidates). LLM first token: ~1-3s (network + model). Total time to first token: ~1.5-3.5s. Total stream: ~5-15s depending on answer length.

42. **Q: How does the `_normalize` function in postgres.py handle different PostgreSQL URL formats?**
    > It accepts `postgresql://`, `postgres://` (AWS RDS style), and `postgresql+asyncpg://`. It normalizes all to `postgresql+asyncpg://`. It strips `sslmode` and `channel_binding` (libpq-only params that asyncpg rejects). It enables TLS for non-local hosts. This means the same codebase works with local Docker, Neon, Supabase, and AWS RDS without config changes.

43. **Q: Why does the upload endpoint save the file locally before ingesting?**
    > The ingestion pipeline expects a file path (it uses `fitz.open(path)`, `Document(path)`, etc.). Writing to disk first (data/uploads/) provides: (1) A persistent copy for re-ingestion if needed. (2) Compatibility with the existing parser interface. (3) The ability to compute a SHA-256 file hash for deduplication.

44. **Q: How would you implement confidence-based routing (small model → large model)?**
    > (1) Run retrieval. (2) If best_dense_score > 0.85 (very confident KB match), use a small model (Gemma 3 4B) — the context is so relevant that even a small model can synthesize well. (3) If 0.6 < score < 0.85, use the full model (Llama 3.3 70B). (4) If score < 0.6, fall back to web search + full model. This cuts costs by 60% for high-confidence queries.

45. **Q: What's the tradeoff in Verity's choice of chunk_size=800?**
    > Too small (200): fragments lose context, embeddings are noisy, retrieval requires more chunks. Too large (2000): embeddings are diluted, mixing multiple topics, precision drops. 800 chars (~200 tokens) is within the 512-token limit of BGE-small, captures a full paragraph or subsection, and produces focused embeddings.

46. **Q: How would you add multi-modal retrieval (images + text)?**
    > (1) At ingestion: extract images from PDFs (already implemented). (2) Embed images with a CLIP model. (3) Store image embeddings in a separate Qdrant collection (different vector dimension). (4) At query time: embed query with both text and CLIP models. (5) Search both collections. (6) Return text chunks AND relevant figures.

47. **Q: Why does the RAPTOR summarizer use `temperature=0.3` instead of 0?**
    > Temperature 0 is deterministic — always picks the most likely token. For summarization, slight randomness (0.3) produces more natural, varied language. The summary should capture key ideas, not be a mechanical concatenation. Higher temperatures (>0.7) would introduce factual errors.

48. **Q: How would you handle a query that spans multiple domains?**
    > Currently, the domain filter is a hard filter — you can only query one domain at a time or "all". Improvement: (1) The planner could detect cross-domain queries. (2) Decompose into per-domain sub-queries. (3) Retrieve from each domain. (4) Merge results with RRF. (5) Or: allow users to select multiple domains.

49. **Q: What monitoring would you add for production?**
    > (1) Latency percentiles: p50, p95, p99 per endpoint. (2) Retrieval quality: average dense score, average number of chunks. (3) LLM metrics: tokens/second, error rate, cost per query. (4) Infrastructure: Qdrant memory, Postgres connections, Redis memory. (5) Business: queries per user per day, session length, upload count. (6) Alerting: if faithfulness drops below 0.7 on weekly eval.

50. **Q: If you were to rewrite Verity from scratch today, what would you change?**
    > (1) Use `langchain-text-splitters` only for the recursive splitter — rewrite the rest in pure Python. (2) Add a semantic query cache from day one. (3) Use LangGraph's streaming API instead of a separate streaming path. (4) Add structured output (JSON mode) for all LLM calls to eliminate JSON parsing failures. (5) Use Pydantic models for all inter-component data instead of raw dicts. (6) Add RLS in Postgres for tenant isolation. (7) Implement proper refresh token rotation. (8) Add a feedback mechanism (thumbs up/down) to improve retrieval over time.

---

# H — COMPLETE FILE REFERENCE (One-Liner Each)

---

### Backend Core
| File | Purpose |
|------|---------|
| [`main.py`](file:///Users/abhijaat/Desktop/Verity/Verity/main.py) | FastAPI app creation, CORS, router registration, Qdrant init on startup |
| [`backend/core/config.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/core/config.py) | Pydantic settings — all env vars with defaults and descriptions |
| [`backend/core/llm.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/core/llm.py) | OpenAI client singleton pointing at OpenRouter, `provider_kwargs()` helper |
| [`backend/core/security.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/core/security.py) | bcrypt password hashing, JWT creation/verification (HS256, 7-day expiry) |

### Database
| File | Purpose |
|------|---------|
| [`backend/db/postgres.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/db/postgres.py) | SQLAlchemy async engine + session factory, URL normalization for Neon/Docker/RDS |
| [`backend/db/qdrant_client.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/db/qdrant_client.py) | Qdrant client + `init_collection()` with payload indexes on user_id, domain |
| [`backend/db/models/user.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/db/models/user.py) | User model: id, email (unique+indexed), hashed_password, is_active |
| [`backend/db/models/document.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/db/models/document.py) | Document model: user_id FK, title, source_path, file_hash, domain, status |
| [`backend/db/models/chunk.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/db/models/chunk.py) | Chunk model: document_id FK, text, chunk_index, qdrant_id, search_vector (TSVECTOR) |
| [`backend/db/models/conversation.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/db/models/conversation.py) | ConversationHistory: session_id+user_id indexed, role, content, created_at |

### Agents (LangGraph)
| File | Purpose |
|------|---------|
| [`backend/agents/graph.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/agents/graph.py) | StateGraph definition — 8 nodes, 2 conditional edges, `build_graph()` factory |
| [`backend/agents/state.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/agents/state.py) | AgentState TypedDict — 22 fields flowing through the graph |
| [`backend/agents/nodes.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/agents/nodes.py) | All node functions: planner, retriever, grader, contradiction, generator, citation, hallucination |

### Ingestion
| File | Purpose |
|------|---------|
| [`backend/ingestion/pipeline.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/pipeline.py) | 7-step pipeline orchestrator: detect → hash → parse → clean → chunk → embed → store |
| [`backend/ingestion/parsers/pdf_parser.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/parsers/pdf_parser.py) | PyMuPDF text extraction + optional vision LLM image descriptions |
| [`backend/ingestion/parsers/docx_parser.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/parsers/docx_parser.py) | python-docx with heading-based section splitting |
| [`backend/ingestion/parsers/web_parser.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/parsers/web_parser.py) | trafilatura web content extraction |
| [`backend/ingestion/parsers/txt_parser.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/parsers/txt_parser.py) | Plain text/markdown file reader |
| [`backend/ingestion/cleaning/text_cleaner.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/cleaning/text_cleaner.py) | Regex-based noise removal: emails, DOIs, references, control chars |
| [`backend/ingestion/chunking/recursive_chunker.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/chunking/recursive_chunker.py) | LangChain RecursiveCharacterTextSplitter: 800 chars, 150 overlap |
| [`backend/ingestion/chunking/heading_chunker.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/chunking/heading_chunker.py) | Keeps heading-based sections as-is, splits if >800 chars |
| [`backend/ingestion/chunking/markdown_chunker.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/chunking/markdown_chunker.py) | LangChain MarkdownTextSplitter respecting # headings |
| [`backend/ingestion/chunking/semantic_chunker.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/chunking/semantic_chunker.py) | Currently delegates to RecursiveChunker (planned: embedding-based topic detection) |
| [`backend/ingestion/chunking/factory.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/chunking/factory.py) | Maps doc_type → chunker class |
| [`backend/ingestion/embedding/embedding_service.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/embedding/embedding_service.py) | fastembed BAAI/bge-small-en-v1.5 wrapper: embed() and embed_one() |
| [`backend/ingestion/graph/entity_extractor.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/graph/entity_extractor.py) | LLM-based entity extraction from chunk text (max 10, lowercase) |
| [`backend/ingestion/raptor/summarizer.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/raptor/summarizer.py) | K-Means clustering → LLM summarization → level-1 Qdrant nodes |
| [`backend/ingestion/multimodal/image_describer.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/ingestion/multimodal/image_describer.py) | Vision LLM (Llama 4 Scout) describes PDF figures as text |

### Retrieval
| File | Purpose |
|------|---------|
| [`backend/retrieval/dense_retriever.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/retrieval/dense_retriever.py) | Qdrant cosine search with user_id/domain payload filters |
| [`backend/retrieval/sparse_retriever.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/retrieval/sparse_retriever.py) | PostgreSQL FTS: tsvector + ts_rank + plainto_tsquery |
| [`backend/retrieval/hybrid_retriever.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/retrieval/hybrid_retriever.py) | Orchestrates dense+sparse+graph, RRF fusion, cross-encoder rerank |
| [`backend/retrieval/hyde.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/retrieval/hyde.py) | HyDE: LLM generates hypothetical answer → embed for retrieval |
| [`backend/retrieval/graph_expander.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/retrieval/graph_expander.py) | Entity-overlap retrieval: extract query entities → MatchAny filter |
| [`backend/retrieval/reranker.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/retrieval/reranker.py) | Cross-encoder (ms-marco-MiniLM) reranking via fastembed ONNX |
| [`backend/retrieval/context_assembler.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/retrieval/context_assembler.py) | Format chunks as numbered `[1] ... [2] ...` context string |
| [`backend/retrieval/web_searcher.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/retrieval/web_searcher.py) | Tavily API → DuckDuckGo fallback → returns [{title, url, content}] |

### API
| File | Purpose |
|------|---------|
| [`backend/api/routes/auth.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/api/routes/auth.py) | /register, /login, /me endpoints |
| [`backend/api/routes/research.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/api/routes/research.py) | /upload, /ingest, /query, /stream, /sessions endpoints (467 lines) |
| [`backend/api/schemas/auth.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/api/schemas/auth.py) | Pydantic: RegisterRequest, LoginRequest, TokenResponse, UserOut |
| [`backend/api/schemas/research.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/api/schemas/research.py) | Pydantic: QueryRequest, IngestRequest, SourceItem, QueryResponse |
| [`backend/api/dependencies/auth.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/api/dependencies/auth.py) | FastAPI dependency: decode JWT → load User → inject into handler |
| [`backend/api/middleware/rate_limit.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/api/middleware/rate_limit.py) | Redis sliding counter: 20 req/20 min per user, graceful degradation |

### Workers, Memory, Prompts, Evaluation
| File | Purpose |
|------|---------|
| [`backend/workers/celery_app.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/workers/celery_app.py) | Celery config: Redis broker/backend, JSON serialization, TLS for managed Redis |
| [`backend/workers/tasks.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/workers/tasks.py) | Two tasks: ingest_document_task (file path), ingest_text_task (raw text from web) |
| [`backend/memory/conversation_store.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/memory/conversation_store.py) | Two-tier memory: in-memory deque(maxlen=10) + Postgres persistence |
| [`backend/prompts/report.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/prompts/report.py) | Generator prompt template: context + question + history → structured answer prompt |
| [`backend/evaluation/evaluator.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/evaluation/evaluator.py) | LLM-as-judge: 4 metrics (faithfulness, relevancy, recall, precision) |
| [`backend/evaluation/schemas.py`](file:///Users/abhijaat/Desktop/Verity/Verity/backend/evaluation/schemas.py) | Dataclasses: EvalSample (Q + ground truth), EvalResult (4 scores) |

### Frontend
| File | Purpose |
|------|---------|
| [`frontend/app/page.tsx`](file:///Users/abhijaat/Desktop/Verity/Verity/frontend/app/page.tsx) | Main chat UI: messages, SSE streaming, sidebar, sources, pipeline badges (655 lines) |
| [`frontend/app/login/page.tsx`](file:///Users/abhijaat/Desktop/Verity/Verity/frontend/app/login/page.tsx) | Login form with auth context integration |
| [`frontend/app/signup/page.tsx`](file:///Users/abhijaat/Desktop/Verity/Verity/frontend/app/signup/page.tsx) | Registration form |
| [`frontend/app/ingest/page.tsx`](file:///Users/abhijaat/Desktop/Verity/Verity/frontend/app/ingest/page.tsx) | Document upload (drag-drop) + URL ingestion with async task polling |
| [`frontend/components/upload-modal.tsx`](file:///Users/abhijaat/Desktop/Verity/Verity/frontend/components/upload-modal.tsx) | In-chat upload modal: file select → title → domain → upload & start chat |
| [`frontend/lib/api.ts`](file:///Users/abhijaat/Desktop/Verity/Verity/frontend/lib/api.ts) | API client: auth headers, SSE streaming, all backend calls (269 lines) |
| [`frontend/lib/auth-context.tsx`](file:///Users/abhijaat/Desktop/Verity/Verity/frontend/lib/auth-context.tsx) | React context: login/register/logout, token in localStorage, auto-validate on mount |
| [`frontend/lib/auth.ts`](file:///Users/abhijaat/Desktop/Verity/Verity/frontend/lib/auth.ts) | getToken/setToken/clearToken from localStorage |
| [`frontend/app/globals.css`](file:///Users/abhijaat/Desktop/Verity/Verity/frontend/app/globals.css) | Dark theme CSS variables, custom scrollbar, Tailwind config |

### Scripts
| File | Purpose |
|------|---------|
| [`scripts/arxiv_ingest.py`](file:///Users/abhijaat/Desktop/Verity/Verity/scripts/arxiv_ingest.py) | CLI: search arXiv → download PDFs → ingest into Verity |
| [`scripts/build_raptor.py`](file:///Users/abhijaat/Desktop/Verity/Verity/scripts/build_raptor.py) | CLI: build RAPTOR summary nodes from existing chunks |
| [`scripts/run_eval.py`](file:///Users/abhijaat/Desktop/Verity/Verity/scripts/run_eval.py) | CLI: run LLM-as-judge evaluation (baseline vs reranker, 20 golden questions) |

### Infrastructure
| File | Purpose |
|------|---------|
| [`docker-compose.yml`](file:///Users/abhijaat/Desktop/Verity/Verity/docker-compose.yml) | 6-service deployment: postgres, qdrant, redis, backend, celery, frontend |
| [`Dockerfile`](file:///Users/abhijaat/Desktop/Verity/Verity/Dockerfile) | Python 3.12 slim image for backend + celery worker |
| [`.env.example`](file:///Users/abhijaat/Desktop/Verity/Verity/.env.example) | Template of all required environment variables |
