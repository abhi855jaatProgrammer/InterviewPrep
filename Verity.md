# Verity — Complete Interview Preparation Guide
**Role Target:** Software Engineer / Full Stack Developer / AI Backend Engineer  
**Target Companies:** Google, Amazon, Microsoft, Uber, Atlassian, Flipkart, Top Tech Product Companies  
**Project:** Verity — Production-Grade Agentic RAG & Knowledge Synthesis Platform  
**Language Level:** Simple, Clear, Professional, Interview-Ready English (Easy to speak & remember)

---

## 1. Project Name Explanation

### Why this name was chosen & what it represents:
* **Literal Meaning:** The word **"Verity"** comes from Latin (*veritas*), which means **"Truth"**, **"Reality"**, or **"A true principle or belief"**.
* **Why Chosen:** Large Language Models (LLMs) often hallucinate and invent fake facts. The core mission of this project is to guarantee **factual correctness, verified truth, and zero hallucinations** when synthesizing answers from research papers.
* **Business & Technical Meaning:** In enterprise research and decision-making, an unverified AI answer is dangerous. "Verity" represents an AI system with built-in verification, citation checking, and multi-source truth validation.

---

### Simple Version (30 seconds)
> "The project is called **Verity**, which means **'Truth'**. I chose this name because regular AI models often hallucinate and make up facts. Verity is built to guarantee truth. It searches through research papers, finds real facts, double-checks citations, and ensures every single sentence generated is 100% grounded in verified source documents."

---

### Professional Interview Version (1 minute)
> "The name of my project is **Verity**, which stands for truth and verifiable knowledge. When professionals read complex research papers, they cannot afford inaccurate AI summaries. 
> 
> Standard RAG systems blindly pass retrieved text to an LLM. Verity was designed from the ground up to solve this trust issue. It uses hybrid retrieval, cross-encoder reranking, and specialized verification agents that validate every citation and detect cross-document contradictions before presenting the final answer. The name reflects our core architectural promise: **every generated response represents verifiable truth backed by source citations.**"

---

### Advanced Version (2 minutes)
> "The project is named **Verity**, derived from the Latin word for truth. In modern AI systems, the biggest bottleneck to real-world adoption is hallucination and epistemic uncertainty. 
> 
> Technically and architecturally, 'Verity' represents three core pillars:
> 1. **Retrieval Truth:** Using Hybrid Search—combining dense semantic embeddings via Qdrant and sparse BM25 keyword matching via PostgreSQL full-text search—fused by Reciprocal Rank Fusion (RRF) and rescored by an ONNX Cross-Encoder so only truly relevant context reaches the LLM.
> 2. **Multi-Agent Verification:** Employing LangGraph agents that act as independent fact-checkers, contradiction detectors, and citation verifiers.
> 3. **Closed-Loop Knowledge Expansion:** If confidence is below threshold, an adaptive agent fetches live web data, feeds the response, and asynchronously embeds it into the user's private knowledge base via Celery.
> 
> Thus, Verity is not just a search tool; it is a self-verifying, continuous knowledge synthesis platform."

---

## 2. Project Introduction

### What is the project?
Verity is an open-source, production-grade **Agentic Retrieval-Augmented Generation (RAG) platform**. It allows researchers, engineers, and students to upload documents (PDFs, DOCX, Markdown, Web URLs) and ask complex domain questions. Instead of reading dozens of 20-page papers, users receive concise, factual, multi-source answers complete with inline bracketed citations `[1][2]` and confidence metrics.

### Who uses it?
* **AI Researchers & Graduate Students:** To synthesize literature reviews across dozens of ArXiv papers.
* **Software Engineers & Architects:** To query technical documentation, RFCs, and API specifications.
* **Enterprise Analysts:** To verify claims across multi-page business and legal reports without hallucination risks.

### What problem does it solve?
* Reading 20+ dense papers manually takes hours or days.
* Standard ChatGPT / LLMs hallucinate, lack access to private documents, and do not provide verifiable paragraph-level citations.
* Basic RAG (simple vector search + prompt) misses exact keywords, fails on broad summary questions, and cannot detect when two papers contradict each other.

### Why is it important?
Verity bridges the gap between raw data storage and high-trust AI reasoning. It demonstrates how to combine **Vector Databases (Qdrant)**, **Relational Databases (PostgreSQL)**, **FastAPI**, **Next.js 16**, and **LangGraph Multi-Agent Workflows** into a secure, multi-tenant system.

---

### Simple English Version
> "Verity is an intelligent assistant for research papers. You upload your research papers or PDFs, and you can ask any question. Verity searches your papers, finds the exact relevant paragraphs, checks for contradictions, and writes an easy-to-understand answer with exact citations like `[1]` and `[2]`."

### Interview Version
> "Verity is a full-stack, production-ready Agentic RAG platform designed to eliminate hallucinations in academic and technical document synthesis. 
> 
> On the frontend, it features a modern Next.js 16 chat interface with real-time Server-Sent Events (SSE) streaming. On the backend, it is powered by FastAPI, Python 3.12, Qdrant vector database, and PostgreSQL. It uses a LangGraph multi-agent architecture where autonomous agents handle query planning, hybrid retrieval with cross-encoder reranking, contradiction detection, citation verification, and automated web fallback."

### Elevator Pitch (30 seconds)
> "Imagine having a research assistant who reads 50 research papers in seconds, extracts the exact facts you need, flags any conflicting information between authors, and writes a verified summary where every claim links to the exact source page. That is **Verity**—an Agentic RAG system built for high-accuracy document intelligence."

---

### Detailed Explanation (2 minutes)
> "Let me walk you through how Verity works end-to-end:
> 
> When a user uploads a document like a PDF or Word file, our ingestion pipeline cleans the text, parses figures using vision LLMs, chunks the content intelligently, generates 384-dimensional dense embeddings locally via FastEmbed ONNX, and indexes both dense vectors in Qdrant and sparse text in PostgreSQL with GIN indexes.
> 
> When the user queries the system, we support two distinct execution paths:
> 1. **Streaming Interactive Path (`/research/query/stream`):** Built for ultra-low latency in the chat UI. It uses fast hybrid retrieval (dense cosine + BM25 tsvector) fused by Reciprocal Rank Fusion, followed by a cross-encoder reranker, immediately streaming tokens via SSE.
> 2. **Full Agentic Path (`/research/query`):** Orchestrated with LangGraph for deep research. A **Planner Agent** breaks the query into sub-questions and applies Hypothetical Document Embeddings (HyDE). A **Grader Agent** inspects retrieved chunks; if relevance is insufficient, it triggers a **Web Search Agent**. A **Contradiction Detector** identifies conflicting facts between papers, the **Generator** synthesizes the answer, and a **Citation Verifier & Hallucination Checker** validates every reference before completion.
> 
> The platform includes JWT authentication, complete per-user knowledge base isolation, Celery background workers with Redis, and a custom LLM-as-a-judge evaluation harness that proved a **41% improvement in context recall** using our cross-encoder reranker."

---

### Possible Follow-Up Questions and Answers

#### Q1: "Why do you need both a streaming path and a full agentic path?"
* **Answer:** "In real-world production, user experience depends on Time-to-First-Token (TTFT). Running 5 sequential LLM grading and verification agents adds 3 to 6 seconds of latency. For live chat, users want immediate streaming feedback. So the streaming path runs fast hybrid retrieval + reranking + token streaming, while the deep research path executes the full LangGraph multi-agent verification pipeline."

#### Q2: "What happens if the user asks a question not in the uploaded documents?"
* **Answer:** "Our hybrid retriever checks the cosine similarity score. If the best score is below 0.45, the system automatically falls back to an integrated Web Search Agent (Tavily/DuckDuckGo). It synthesizes an immediate answer and fires an asynchronous Celery task to clean, embed, and store the web findings into the user's private knowledge base for future queries."

---

## 3. Problem Statement

### The Real-World Situation:
Before building Verity, analyzing large collections of research papers or technical documentation had five critical pain points affecting researchers, developers, and knowledge workers.

---

### Problem 1: Information Overload & Manual Reading Fatigue
* **Description:** Reading twenty 15-page academic papers to answer one research question takes 10–15 hours.
* **Impact:** High engineering time wasted, slow decision making, and missed critical insights.
* **Beginner Explanation:** People do not have time to read hundreds of pages just to find one answer.
* **Interview Explanation:** High cognitive load and manual document inspection create an information processing bottleneck in fast-moving domains like AI and medicine.
* **Business Explanation:** High-value knowledge workers spend up to 30% of their working hours searching for internal documentation rather than executing tasks.

---

### Problem 2: LLM Hallucinations & Lack of Trust
* **Description:** Standard LLMs (like raw ChatGPT) generate convincing answers that contain fabricated citations, false equations, or imaginary paper authors.
* **Impact:** Inaccurate summaries lead to flawed engineering decisions or invalid academic research.
* **Beginner Explanation:** AI often makes up fake information that sounds completely real.
* **Interview Explanation:** Parametric LLM memory suffers from hallucination and lack of temporal grounding, rendering unaugmented LLM outputs unsuitable for enterprise compliance.
* **Business Explanation:** Zero tolerance for unverified data in legal, medical, and technical research contexts where wrong facts create massive liabilities.

---

### Problem 3: The Failure of "Naive Vector Search" (Keyword & Vocabulary Mismatch)
* **Description:** Basic RAG only uses vector cosine similarity. If a user searches for an exact term (e.g., error code `"ERR_403_AUTH"` or specific author name `"Vaswani 2017"`), dense embeddings often fail because they focus on semantic meaning rather than exact keywords.
* **Impact:** The system returns irrelevant chunks and misses the exact paragraph containing the answer.
* **Beginner Explanation:** Regular AI search forgets exact words and numbers, searching only for similar general ideas.
* **Interview Explanation:** Pure dense semantic retrieval suffers from vocabulary mismatch and fails on exact token matches, acronyms, and rare technical identifiers.
* **Business Explanation:** Decreased search recall leads to user frustration and abandonment of internal AI tools.

---

### Problem 4: Conflicting Claims Across Different Papers
* **Description:** Paper A (2022) might say *"Method X achieves 80% accuracy"*, while Paper B (2024) says *"Method X fails on noisy datasets"*. Naive RAG blends both chunks into one prompt, confusing the LLM and generating contradictory nonsense.
* **Impact:** Users receive misleading, blended answers without knowing which source said what.
* **Beginner Explanation:** When two books disagree, simple AI gets confused and combines them into a messy lie.
* **Interview Explanation:** Standard RAG pipelines lack multi-source dispute detection, resulting in epistemic synthesis errors.
* **Business Explanation:** Leaders receive compromised intelligence that conceals critical risks and debates in technical literature.

---

### Problem 5: Missing Context & Cold-Start Knowledge Gaps
* **Description:** If a user's uploaded library does not contain the answer, standard RAG says *"I do not know"* or hallucinates.
* **Impact:** Dead-end user experience requiring the user to leave the app, search Google, download PDFs, and re-upload.
* **Beginner Explanation:** If the document doesn't have the answer, old systems just fail.
* **Interview Explanation:** Static knowledge bases lack adaptive self-healing capabilities when encountering out-of-domain queries.
* **Business Explanation:** Poor user retention and workflow disruption caused by friction in manual knowledge curation.

---

## 4. Why Did You Build This Project?

### Personal Motivation:
> "I built Verity because I wanted to move beyond toy AI tutorials and build a real, production-grade system. Most online tutorials show basic LangChain vector search with 5 lines of code, but they fail in real production. I wanted to master the full engineering stack: custom chunking, hybrid retrieval, cross-encoder rerankers, multi-agent state machines with LangGraph, async queues with Celery, and multi-tenant security."

### Technical Motivation:
* To master **FastAPI** with asynchronous concurrency (`asyncpg`, streaming responses).
* To understand **Hybrid Search** by implementing **Reciprocal Rank Fusion (RRF)** and **Cross-Encoders** locally using ONNX FastEmbed (zero PyTorch overhead).
* To build autonomous multi-agent pipelines with **LangGraph** (cyclic graphs, state reducers, dynamic routing).
* To implement modern frontend architecture using **Next.js 16**, **TypeScript**, and **Server-Sent Events (SSE)**.
* To create a rigorous **LLM-as-a-judge evaluation harness** to measure actual quantitative performance.

### Business Motivation:
* Modern enterprises need private, secure AI workspaces where sensitive documents never leak across user boundaries.
* Creating an "Adaptive Knowledge Base" that automatically queries the web on low-confidence hits and ingests data in the background transforms a static tool into an autonomous knowledge engine.

---

### Interview Answer: "Why did you build this project?"

#### 30-Second Answer:
> "I built Verity to solve the trust and accuracy problems in AI search. While standard RAG systems rely on simple vector search that often hallucinates or misses exact keywords, Verity implements a complete production pipeline: hybrid dense-sparse search, cross-encoder reranking, and LangGraph verification agents. I wanted to build a high-performance, full-stack system that delivers mathematically grounded, citation-verified answers."

#### 1-Minute Answer:
> "I built Verity because I wanted to solve the fundamental limitations of modern RAG systems. In real-world technical research, standard vector search fails on exact keyword lookups, broad cross-document summaries, and contradictory sources.
> 
> I engineered Verity as a full-stack solution: Next.js 16 on the frontend with streaming SSE, and FastAPI with Python 3.12 on the backend. I implemented a hybrid retrieval engine combining Qdrant vector search and PostgreSQL full-text search fused by Reciprocal Rank Fusion, with local ONNX cross-encoder reranking. To guarantee accuracy, I orchestrated LangGraph agents to detect contradictions and verify citations. It was an opportunity to master distributed systems, vector databases, multi-agent design, and rigorous LLM evaluation."

#### 2-Minute Answer:
> "My motivation for building Verity came from observing the huge gap between demo-level AI apps and production-grade software engineering.
> 
> Most RAG implementations suffer from three major engineering problems:
> 1. **Retrieval Degradation:** Dense-only embeddings lose exact keywords, numbers, and technical terms. I solved this by building a hybrid retrieval engine combining Qdrant for semantic search and PostgreSQL tsvector with GIN indexing for sparse search, fused via Reciprocal Rank Fusion and rescored using an ONNX cross-encoder.
> 2. **Lack of Agentic Verification:** Standard systems blindly trust whatever the LLM outputs. I designed a multi-agent state machine using LangGraph that performs query decomposition with HyDE, grades retrieved context, checks for inter-document contradictions, and verifies every bracketed citation before the user sees it.
> 3. **Enterprise Readiness & Latency:** In production, you need user isolation and fast responses. I implemented JWT auth with per-user knowledge base isolation, Celery/Redis for background ingestion, and separated the architecture into a low-latency SSE streaming path for chat and a full agentic graph for deep research.
> 
> Finally, I proved its effectiveness with an automated evaluation harness, demonstrating a **41% increase in context recall** over standard RRF baselines."

---

## 5. Solution Overview

| Problem in Standard Systems | Verity Solution | Tangible Engineering Benefit |
| :--- | :--- | :--- |
| **Missing exact keywords / IDs** | **Hybrid Retrieval (Dense Qdrant + Sparse Postgres FTS)** | Captures both deep semantic concepts and exact keyword matches (e.g. acronyms, error codes). |
| **Poor retrieval ranking / noise** | **Cross-Encoder Reranker (ms-marco-MiniLM via ONNX)** | Jointly scores query-document relevance; improves Context Recall by **+41%** and Precision by **+16%**. |
| **Vague or short user queries** | **HyDE (Hypothetical Document Embeddings) & Planner Agent** | Expands raw queries into hypothetical technical answers before embedding, finding better vector matches. |
| **Conflicting data between sources** | **Contradiction Detector Node (LangGraph)** | Explicitly highlights when two papers disagree instead of blending fake compromises. |
| **Hallucinated citations** | **Citation Verifier & Hallucination Checker** | Validates that claim `[1]` maps to text in Chunk `[1]` and verifies factual grounding against source context. |
| **Knowledge gaps on missing data** | **Adaptive Web Fallback & Background Auto-Ingest** | Triggers Tavily/DuckDuckGo when similarity < 0.45, answers instantly, and queues Celery ingestion for next time. |
| **Slow blocking document ingestion** | **Asynchronous Celery Task Queue + Redis** | Heavy PDF parsing, OCR vision extraction, and embedding happen in background workers without blocking the API. |
| **Data security & multi-tenancy** | **JWT Auth + Row/Payload Tenant Scoping** | Every Postgres row and Qdrant vector payload is tagged with `user_id`, preventing cross-user data leakage. |

---

### Before vs. After Verity

```
BEFORE VERITY (Standard / Naive RAG):
[Query] -> [Dense Vector Search] -> [Top 5 Chunks] -> [LLM Generates Text]
Result: Missed exact keywords, included irrelevant chunks, hallucinated fake citations, 
        confused by contradictions, and took 10+ seconds with no streaming.

AFTER VERITY (Agentic Production RAG):
[Query] -> [Planner/HyDE] -> [Hybrid Search: Qdrant + Postgres FTS] -> [RRF Fusion] 
        -> [ONNX Cross-Encoder Rerank] -> [Grader Check / Web Fallback] 
        -> [Contradiction Check] -> [Streaming LLM + Citation Verification]
Result: +41% higher recall, verified inline citations, instant token streaming via SSE, 
        and automatic self-learning from web fallbacks.
```

### Measurable Improvements (from our Evaluation Harness):
* **Context Recall:** Increased from **0.32 to 0.45 (+41% relative gain)** by introducing cross-encoder reranking.
* **Context Precision:** Increased from **0.63 to 0.73 (+16% relative gain)**.
* **Faithfulness Score:** Increased to **0.89 (89% verified factual grounding)**.
* **Time-To-First-Token (TTFT):** Reduced to **< 600ms** on the streaming query endpoint.

---

## 6. System Architecture

### High-Level Architecture

```
                       ┌────────────────────────────────────────┐
                       │           Next.js 16 Frontend          │
                       │   (React 19, Tailwind, shadcn/ui, SSE) │
                       └───────────────────┬────────────────────┘
                                           │ HTTPS / SSE Stream
                                           ▼
                       ┌────────────────────────────────────────┐
                       │          FastAPI Backend Engine        │
                       │   (Python 3.12, Uvicorn, AsyncPG, JWT) │
                       └─────┬──────────────┬──────────────┬────┘
                             │              │              │
        ┌────────────────────┴───┐          │          ┌───┴────────────────────┐
        │                        │          │          │                        │
        ▼                        ▼          ▼          ▼                        ▼
┌───────────────┐        ┌──────────────┐ ┌───┐ ┌───────────────┐        ┌──────────────┐
│ Qdrant Vector │        │ PostgreSQL   │ │ R │ │ Celery Task   │        │ OpenRouter   │
│ Database      │        │ Relational + │ │ e │ │ Workers       │        │ LLM / Vision │
│ (384-dim BGE) │        │ FTS (GIN)    │ │ d │ │ (Background)  │        │ (Llama 3.3   │
└───────────────┘        └──────────────┘ │ i │ └───────────────┘        │  70B / 4V)   │
                                          │ s │                          └──────────────┘
                                          └───┘
```

#### Component Responsibilities:
1. **Frontend (Next.js 16 / React):** Renders the responsive chat interface, handles Markdown & LaTeX math rendering, manages auth state, and consumes token-by-token SSE streaming.
2. **Backend API (FastAPI):** Exposes async REST endpoints, handles JWT auth & password hashing (bcrypt), validates payloads via Pydantic, and manages database sessions.
3. **Hybrid Retriever Layer:** Connects to Qdrant (dense cosine vector search) and PostgreSQL (sparse full-text search with `tsvector`), combines candidates using Reciprocal Rank Fusion ($k=60$), and reranks via a local ONNX cross-encoder.
4. **Agent Orchestration (LangGraph):** Manages cyclic and conditional agent graphs (Planner, Grader, Web Search, Contradiction Detector, Generator, Citation Verifier, Hallucination Checker).
5. **Worker Layer (Celery + Redis):** Asynchronously parses multi-page PDFs, extracts figures for vision analysis, cleans text, and embeds vectors without blocking HTTP threads.
6. **Persistence Layer:** PostgreSQL stores users, documents, chunks, and conversation histories. Qdrant stores dense embeddings and payload filters.

---

### Step-by-Step Request Flow (Streaming Path)

```
[1. User types query in Next.js UI]
       │
[2. Frontend attaches JWT Bearer token and opens POST /research/query/stream]
       │
[3. FastAPI verifies JWT, extracts user_id, and checks per-user rate limit (Redis)]
       │
[4. Hybrid Retriever executes concurrent search:
       ├─ Qdrant: Top-20 Dense Vector matches (user_id scoped)
       └─ PostgreSQL: Top-20 Sparse BM25 tsvector matches (user_id scoped)]
       │
[5. RRF Fusion combines both lists -> Top 30 candidates]
       │
[6. Local ONNX Cross-Encoder rescores and outputs Top 8 most relevant chunks]
       │
[7. Fallback Check: Is best dense cosine score >= 0.45?
       ├─ If YES: Use retrieved chunks as context
       └─ If NO: Call Web Search Agent, stream web context, and enqueue Celery ingest task]
       │
[8. FastAPI streams LLM tokens via Server-Sent Events (SSE) to Frontend]
       │
[9. Frontend renders tokens in real time; Backend saves conversation turn to Postgres]
```

---

### Data Flow (End-to-End Life of a Document)

1. **Input:** User uploads a PDF (`/research/upload`).
2. **Deduplication:** Backend calculates SHA-256 hash. If hash exists for `user_id`, instant skip.
3. **Processing & Cleaning:** PyMuPDF extracts raw text and images. Cleaning pipeline strips email headers, DOIs, page numbers, and copyright noise.
4. **Chunking:** Document is chunked using recursive structural boundaries (512 tokens with 50-token overlap).
5. **Embedding:** Local FastEmbed engine runs `bge-small-en-v1.5` ONNX model to generate 384-dimensional dense vectors.
6. **Storage:**
   * Text chunks, metadata, and search vectors (`tsvector`) stored in PostgreSQL.
   * Embeddings stored in Qdrant collections tagged with payload: `{user_id, doc_id, domain}`.
7. **Retrieval & Output:** When queried, relevant chunks are retrieved, fused, reranked, and formatted into prompt context for LLM generation.

---

## 7. Frontend Deep Dive

### Why Frontend Was Needed
A technical AI tool needs an intuitive UI:
* Real-time streaming output (users shouldn't stare at a spinner for 5 seconds).
* Multi-session chat history and document library management.
* Highlighting source citations and confidence metrics.
* Direct in-chat PDF drag-and-drop file upload.

### Technologies Used

| Technology | Why Chosen | Alternatives Considered | Advantages | Disadvantages / Tradeoffs |
| :--- | :--- | :--- | :--- | :--- |
| **Next.js 16 (App Router)** | Industry standard for React, excellent SSR/SSG, fast routing. | Vite + React SPA, Remix | Seamless server components, clean directory-based routing, SEO ready. | Steeper learning curve than simple Vite SPA. |
| **TypeScript** | Type safety across API contracts and chat state. | Vanilla JavaScript | Catches runtime null/undefined bugs during compilation. | Requires writing extra interfaces and types. |
| **Tailwind CSS** | Fast, utility-first styling with zero CSS file clutter. | CSS Modules, Styled Components | Rapid prototyping, consistent spacing/color system, small bundle. | Can clutter JSX markup with long class strings. |
| **shadcn/ui** | Accessible, headless UI components built on Radix primitives. | Material UI, Ant Design | Full code ownership (copy-paste in project), highly customizable. | Not an installable npm library; files live in codebase. |
| **Server-Sent Events (SSE)** | Lightweight, unidirectional HTTP streaming for token generation. | WebSockets, Long Polling | Native browser `EventSource` / Fetch stream support, reconnects automatically, runs over standard HTTP/2. | Unidirectional only (client cannot send stream over same socket). |
| **Lucide Icons** | Clean, modern, lightweight SVG icons. | FontAwesome, Material Icons | Tree-shakeable, beautiful modern aesthetic. | None. |

---

### Key Frontend Interview Questions & Answers

#### Q1: "Why did you choose Server-Sent Events (SSE) instead of WebSockets for streaming answers?"
* **Answer:** "For an AI chat interface, communication during answer generation is strictly **unidirectional** (the server streams tokens to the client). WebSockets introduce unnecessary complexity: stateful connection management, custom ping/pong heartbeats, and firewall/proxy issues. SSE runs over standard HTTP/1.1 or HTTP/2, uses standard HTTP headers for authentication (Bearer JWT), and supports built-in reconnection logic with minimal client-side code."

#### Q2: "How did you handle the chunked SSE stream on the frontend in React?"
* **Answer:** "I used the browser's `fetch` API with `response.body.getReader()`. As binary chunks arrived over the wire, I decoded them using a `TextDecoderStream` into JSON events (e.g., `event: token`, `event: sources`, `event: done`). I updated the React message state incrementally using functional state updates (`setMessages(prev => ...)`) to prevent race conditions and ensure smooth 60fps rendering without re-rendering the entire chat tree."

---

## 8. Backend Deep Dive

### Backend Responsibilities
* Expose secure, validated REST & SSE endpoints.
* Manage multi-tenant JWT authentication, password hashing, and user sessions.
* Execute asynchronous database queries using SQLAlchemy 2.0 with `asyncpg`.
* Orchestrate multi-agent RAG pipelines using LangGraph.
* Manage background Celery tasks with Redis broker.

### Technologies Used

| Technology | Why Chosen | Alternatives Considered | Advantages | Disadvantages / Tradeoffs |
| :--- | :--- | :--- | :--- | :--- |
| **FastAPI** | Modern, ultra-fast Python web framework with native async support. | Flask, Django | Native OpenAPI docs (`/docs`), Pydantic validation, asynchronous speed. | Requires careful async/sync boundary management. |
| **Python 3.12** | Latest standard for AI/ML ecosystems, optimized interpreter speed. | Node.js, Go | Rich AI libraries (FastEmbed, LangGraph, PyMuPDF). | Slower raw CPU execution than Go/Rust for pure compute. |
| **LangGraph** | Cyclic graph-based agent orchestration with fine-grained state control. | LangChain Chains, CrewAI | Supports loops, human-in-the-loop, conditional branching, and explicit state reducers. | More complex setup than linear chains. |
| **FastEmbed (ONNX)** | In-process vector embeddings and cross-encoder reranking. | PyTorch + HuggingFace, OpenAI API | Runs locally via ONNX Runtime with zero GPU requirement and no 2GB PyTorch dependency; no external embedding API latency or cost. | Limited to models supported in the FastEmbed ONNX registry. |
| **Celery + Redis** | Industry-standard distributed asynchronous task queue. | RQ, BackgroundTasks | Robust retries, distributed worker support, task monitoring. | Requires running a separate Redis instance and worker process. |

---

### Backend Architecture Components:
* **Authentication:** Stateless JWT tokens signed with `HS256`, 7-day expiration. Passwords hashed using `bcrypt` with automated salt generation.
* **Authorization:** FastAPI dependency injection (`get_current_user`) verifies token on every protected route and extracts `user_id`.
* **Validation:** Pydantic V2 models strictly validate every incoming JSON request and outgoing response schema.
* **Error Handling:** Centralized exception handlers catch custom application errors (e.g., `DocumentNotFoundError`, `RateLimitExceeded`) and return standard RFC 7807 error responses.
* **Logging & Tracing:** Structured Python logging plus optional Langfuse tracing for LLM generation spans, tracking token counts and latency.

---

### Key Backend Interview Questions & Answers

#### Q1: "How did you avoid blocking the FastAPI async event loop during CPU-intensive tasks like PDF parsing or embedding?"
* **Answer:** "FastAPI runs on an async event loop (asyncio). If you execute heavy CPU tasks like PDF text extraction or ONNX embedding inside an `async def` route, you block the entire event loop, freezing all concurrent requests. To prevent this, I used two strategies:
  1. For short CPU operations, I offloaded them using `asyncio.to_thread(sync_function)`.
  2. For large document ingestion and background web indexing, I delegated the work completely to **Celery workers** backed by **Redis**, returning an immediate `202 Accepted` response with a task ID to the user."

#### Q2: "How does LangGraph coordinate the agent workflow?"
* **Answer:** "LangGraph models agent workflows as a directed state graph. We define an explicit `AgentState` TypedDict that holds the user query, retrieved documents, grading status, contradiction flags, and generated answer. Nodes are pure Python functions that take state and return partial state updates. Conditional edges inspect state properties (e.g., `should_web_search()` checks if retrieved chunks are empty or irrelevant) to dynamically route execution to the next node."

---

## 9. Database Design

### Why We Chose PostgreSQL + Qdrant
* **PostgreSQL:** The gold standard for ACID relational data (Users, Documents, Chunks, Chat Sessions) combined with powerful built-in Full-Text Search (`tsvector` + GIN indexing) for sparse BM25 retrieval.
* **Qdrant:** A dedicated, ultra-fast vector database written in Rust. It excels at filtered dense vector search, allowing us to filter by `{user_id: "...", domain: "..."}` at high throughput with low memory overhead.

---

### Relational Schema Design (PostgreSQL)

```
┌─────────────────────────┐         ┌─────────────────────────┐
│          users          │         │      chat_sessions      │
├─────────────────────────┤         ├─────────────────────────┤
│ id (UUID, PK)           │1       *│ id (UUID, PK)           │
│ email (VARCHAR, Unique) ├─────────┤ user_id (UUID, FK)      │
│ hashed_password (TEXT)  │         │ title (VARCHAR)         │
│ created_at (TIMESTAMP)  │         │ created_at (TIMESTAMP)  │
└────────────┬────────────┘         └────────────┬────────────┘
             │1                                  │1
             │                                   │
             │*                                  │*
┌────────────┴────────────┐         ┌────────────┴────────────┐
│        documents        │         │      chat_messages      │
├─────────────────────────┤         ├─────────────────────────┤
│ id (UUID, PK)           │         │ id (UUID, PK)           │
│ user_id (UUID, FK)      │         │ session_id (UUID, FK)   │
│ filename (VARCHAR)      │         │ role (user/assistant)   │
│ file_hash (VARCHAR)     │         │ content (TEXT)          │
│ file_size (INT)         │         │ sources (JSONB)         │
│ created_at (TIMESTAMP)  │         │ created_at (TIMESTAMP)  │
└────────────┬────────────┘         └─────────────────────────┘
             │1
             │
             │*
┌────────────┴────────────┐
│      document_chunks    │
├─────────────────────────┤
│ id (UUID, PK)           │
│ document_id (UUID, FK)  │
│ user_id (UUID, FK)      │
│ chunk_index (INT)       │
│ content (TEXT)          │
│ tsv (TSVECTOR - GIN)    │
│ metadata (JSONB)        │
└─────────────────────────┘
```

---

### Indexing & Query Optimization
1. **Full-Text Search (FTS) Index:**
   ```sql
   CREATE INDEX idx_chunks_tsv ON document_chunks USING GIN(tsv);
   ```
   * Enables sub-millisecond sparse keyword search across hundreds of thousands of chunks using `to_tsquery('english', query)`.
2. **Multi-Tenant User Scoping Index:**
   ```sql
   CREATE INDEX idx_chunks_user_doc ON document_chunks(user_id, document_id);
   CREATE INDEX idx_docs_user_hash ON documents(user_id, file_hash);
   ```
   * Makes deduplication checks and user-isolated queries instantaneous.
3. **Qdrant Vector Indexing:**
   * Uses **HNSW (Hierarchical Navigable Small World)** graphs for 384-dimensional dense vectors with Cosine distance metric.
   * Configures payload schema indexing on `user_id` for pre-filtered vector search.

---

### Key Database Interview Questions & Answers

#### Q1: "Why not just use pgvector inside PostgreSQL instead of a separate Qdrant instance?"
* **Answer:** "While `pgvector` is convenient for small projects, a dedicated vector search engine like Qdrant provides distinct architectural advantages in production:
  1. **Separation of Concerns:** Heavy HNSW vector graph traversals consume significant CPU and RAM. Isolating vector search in Qdrant ensures that complex vector queries never starve relational PostgreSQL transactions.
  2. **Advanced Payload Filtering:** Qdrant performs fast payload filtering alongside HNSW search natively in Rust.
  3. **Independent Scalability:** We can scale vector search nodes independently from relational database instances based on traffic profile."

#### Q2: "How does your full-text search query work in PostgreSQL?"
* **Answer:** "We maintain a generated `tsvector` column on the `document_chunks` table that combines the chunk text and document title. We index this column using a GIN (Generalized Inverted Index). During retrieval, we parse user queries into `tsquery` objects and rank results using `ts_rank_cd()`, returning the top 20 sparse candidates in under 5 milliseconds."

---

## 10. Challenges Faced During Development (Top 10 Challenges)

---

### Challenge 1: The "Keyword Blindness" of Pure Dense Vector Search
* **Root Cause:** When searching for specific error codes (e.g. `CUDA_OUT_OF_MEMORY`) or exact algorithm names, dense embeddings mapped them to generic 'error' concepts, returning irrelevant chunks.
* **Investigation:** Inspected Qdrant retrieval outputs and noticed high cosine distance for exact string queries.
* **Solution:** Implemented **Hybrid Retrieval**. We query both Qdrant (dense) and PostgreSQL FTS (sparse BM25), combine the ranks using **Reciprocal Rank Fusion (RRF)**:
  $$\text{RRF Score} = \sum \frac{1}{60 + \text{rank}}$$
* **What I Learned:** Dense and sparse search are complementary. Semantic search handles meaning; keyword search handles precision.
* **Interview Response:** *"I identified that dense search alone failed on technical identifiers. I solved this by implementing hybrid retrieval with RRF fusion, which combines the strengths of semantic and lexical search."*

---

### Challenge 2: Cross-Encoder Memory & PyTorch Bloat
* **Root Cause:** Standard Hugging Face cross-encoders require PyTorch, adding a 2GB Docker image footprint and slow startup times.
* **Investigation:** Profiled container memory and saw PyTorch consuming > 1.2 GB RAM on idle.
* **Solution:** Replaced PyTorch runtime with **FastEmbed ONNX Runtime**. ONNX models run in C++ with minimal memory footprint (< 150 MB) and execute inference 2.5x faster.
* **What I Learned:** Model quantization and ONNX runtimes are critical for lightweight, cost-efficient production deployments.
* **Interview Response:** *"To prevent high memory usage and huge Docker container sizes, I transitioned our embedding and reranking pipelines to FastEmbed ONNX, cutting memory by 80%."*

---

### Challenge 3: Streaming Tokens with Multi-Agent Verification Latency
* **Root Cause:** Running 5 LangGraph agent nodes (Planner, Grader, Contradiction, Verifier, Hallucination) sequentially added 4–7 seconds of delay before the user saw a single word.
* **Investigation:** Measured latency breakdowns per node; LLM API round-trips were the primary bottleneck.
* **Solution:** Architected a **Dual-Path Pipeline**:
  1. `/research/query/stream`: Optimized for low latency. Runs fast hybrid retrieval + rerank, then streams tokens immediately via SSE.
  2. `/research/query`: Executes the full agentic LangGraph pipeline for deep, asynchronous research.
* **What I Learned:** Production architecture requires balancing deep verification vs. interactive user experience.
* **Interview Response:** *"I bifurcated the architecture into a low-latency SSE streaming path for live chat and a deep agentic verification graph for comprehensive research queries."*

---

### Challenge 4: Multi-User Data Isolation in Vector Search
* **Root Cause:** Standard vector search queries the entire collection, risking cross-tenant data leaks where User A could see User B's private documents.
* **Investigation:** Tested concurrent multi-user queries and noticed vector results returned chunks from other users.
* **Solution:** Added mandatory `user_id` payload tagging on all Qdrant vectors and applied strict payload filter criteria on every retrieval query. Similarly, scoped all PostgreSQL queries by `user_id`.
* **What I Learned:** Multi-tenancy must be enforced at the storage query level, not just in application logic.
* **Interview Response:** *"I enforced strict multi-tenant isolation by tagging every vector payload with `user_id` in Qdrant and applying database-level tenant filters on all queries."*

---

### Challenge 5: Handling Duplicate Document Ingestion
* **Root Cause:** Users frequently upload the same PDF multiple times, leading to duplicate chunks, corrupted retrieval rankings, and wasted vector storage.
* **Investigation:** Monitored database size and noticed identical chunks multiplying in Qdrant.
* **Solution:** Implemented SHA-256 file hashing at the start of ingestion. If a document with the same hash exists for that `user_id`, the system immediately returns the existing document metadata without reprocessing.
* **What I Learned:** Content-addressable hashing is essential for idempotent ingestion pipelines.
* **Interview Response:** *"I implemented SHA-256 content hashing to ensure idempotent file ingestion, completely preventing duplicate chunks and wasted storage."*

---

### Challenge 6: Hallucinated Bracketed Citations `[1][2]`
* **Root Cause:** LLMs would invent citation numbers like `[4]` when only 3 chunks were provided in the prompt context.
* **Investigation:** Evaluated generated outputs with regular expressions and mapped them against input chunk indices.
* **Solution:** Built a deterministic **Citation Verifier** node. It parses all bracketed numbers `[N]` in the generated text, validates them against the actual retrieved context chunks, and strips or flags invalid references.
* **What I Learned:** LLM outputs must always be validated by deterministic post-processing rules.
* **Interview Response:** *"I created a citation verification step that deterministically validates all bracketed references against the ground-truth chunk payload before returning the answer."*

---

### Challenge 7: PDF Formatting Noise & Broken Sentences
* **Root Cause:** Raw academic PDFs contain header lines, page numbers, author email lists, and copyright notices that pollute embeddings.
* **Investigation:** Found that vector search was frequently matching on author affiliations and conference footers instead of paper content.
* **Solution:** Built a multi-stage regex text-cleaning pipeline that strips DOIs, emails, page headers/footers, and joins hyphenated words split across lines before chunking.
* **What I Learned:** Garbage in, garbage out. Text cleaning is 50% of RAG success.
* **Interview Response:** *"I engineered a specialized cleaning pipeline that strips academic noise like DOIs and headers, which significantly boosted retrieval signal-to-noise ratio."*

---

### Challenge 8: Rate Limiting & Abuse Prevention on Free LLM Tiers
* **Root Cause:** Rapid user queries depleted OpenRouter API token limits, crashing concurrent requests.
* **Investigation:** Inspected HTTP 429 errors in backend logs during rapid chat interactions.
* **Solution:** Implemented a Redis-backed sliding-window rate limiter on the streaming endpoint (`2 requests per 20 minutes per IP/User`) and added exponential backoff retries on LLM client calls.
* **What I Learned:** Upstream API failures must be shielded with local rate limiting and graceful retry mechanisms.
* **Interview Response:** *"I integrated a Redis sliding-window rate limiter and client-side retry logic to protect against upstream LLM rate limits and ensure high availability."*

---

### Challenge 9: Memory Leaks & Unclosed SSE Connections
* **Root Cause:** When a user refreshed the browser or closed the tab mid-stream, backend generator loops continued executing, keeping database sessions open.
* **Investigation:** Monitored active PostgreSQL connections and noticed connection pool exhaustion under load.
* **Solution:** Integrated FastAPI `Request.is_disconnected` checks inside the streaming generator loop and wrapped DB sessions in clean `try...finally` context managers.
* **What I Learned:** Streaming endpoints require explicit client disconnect detection to avoid orphan processes and connection leaks.
* **Interview Response:** *"I prevented connection pool exhaustion by actively monitoring client disconnect events during SSE streaming to immediately clean up backend resources."*

---

### Challenge 10: Quantitative Proof of RAG Quality (How to Measure Success)
* **Root Cause:** We needed objective proof that our Reranker and Hybrid search were actually better than baseline vector search.
* **Investigation:** Manual eyeballing of search results was subjective and unscalable.
* **Solution:** Built an automated **LLM-as-a-judge evaluation harness** (`scripts/run_eval.py`). It evaluates 20 curated gold-standard Q/A pairs across 4 metrics: Context Recall, Context Precision, Faithfulness, and Answer Relevancy.
* **What I Learned:** You cannot optimize what you do not measure. Automated evaluation is crucial for iterative AI development.
* **Interview Response:** *"I implemented an automated LLM-as-a-judge evaluation harness that proved our cross-encoder reranker delivered a 41% relative increase in context recall."*

---

## 11. Teamwork and Collaboration

### Solo Project Architecture & Engineering Management
Because this was an intensive solo engineering project, I simulated an agile engineering environment:
1. **Requirements Planning:** Created clear functional specifications and API contracts before writing code.
2. **Architecture Decisions:** Documented trade-offs in architecture decision records (e.g., choosing FastAPI over Django, Qdrant over pgvector).
3. **Task Prioritization:** Built the foundation first (DB schemas + Ingestion) $\rightarrow$ Core Engine (Hybrid Retrieval + LangGraph) $\rightarrow$ Frontend UI $\rightarrow$ Evaluation & DevOps.
4. **Testing & Quality Assurance:** Wrote automated unit tests with `pytest` for parsers, chunkers, and security modules, along with end-to-end evaluation scripts.

---

### Sample Behavioral Interview Answers

#### Tell me about teamwork and how you work with others:
> "In team environments, I prioritize clear communication, well-defined API contracts, and active collaboration. I believe in documenting interfaces with OpenAPI/Swagger early so frontend and backend engineers can work in parallel without blocking each other. I actively participate in code reviews, focusing on code readability, security, and edge-case handling."

#### Tell me about a technical disagreement you had and how you resolved it:
> "In a previous project discussion regarding search architecture, there was a debate between using purely dense vector search versus building a hybrid search pipeline. The argument against hybrid search was the added complexity of managing full-text search indexes. 
> 
> To resolve the disagreement objectively, I proposed an A/B benchmark test. I set up a test dataset with 20 technical queries containing exact error codes and author names. The benchmark proved that dense search failed on 35% of exact keyword queries, while hybrid search achieved 100% recall. By letting data and empirical benchmarks guide the decision, the team aligned on hybrid retrieval."

#### Tell me about a time you took technical leadership:
> "When building this system, I realized that subjective testing of AI responses was leading to inconsistent improvements. I took the initiative to build a comprehensive evaluation harness using the LLM-as-a-judge pattern. I defined four key KPIs—faithfulness, relevancy, context recall, and context precision—and automated report generation. This gave our engineering process clear, quantifiable metrics to validate every prompt and retrieval optimization."

---

## 12. Documentation and Research

### Technical Research & Foundations
* **RAG & Retrieval:** Researched *Lewis et al. (2020)* on Retrieval-Augmented Generation to understand the mathematical foundations of dense retrieval.
* **HyDE (Hypothetical Document Embeddings):** Studied *Gao et al. (2022)* to implement query expansion where the model generates a hypothetical answer before embedding.
* **RAPTOR (Recursive Abstractive Processing for Tree-Organized Retrieval):** Studied *Sarthi et al. (2024)* for recursive clustering of document summaries to answer high-level thematic queries.
* **Reciprocal Rank Fusion (RRF):** Studied *Cormack et al.* to understand non-parametric rank fusion constants ($k=60$).

### Official Documentation Utilized
* **FastAPI Docs:** Used for advanced dependency injection patterns and async generator streaming.
* **LangGraph Documentation:** Utilized for building stateful, multi-agent cyclic graphs with custom reducers.
* **Qdrant Official Docs:** Referenced for HNSW vector index tuning and payload filtering schemas.
* **PostgreSQL Documentation:** Studied full-text search operators (`to_tsvector`, `ts_rank_cd`, and GIN indexing).
* **Next.js App Router Docs:** Referenced for SSE stream handling and server-side route handlers.

---

## 13. Security Considerations

```
┌────────────────────────────────────────────────────────────────────────┐
│                        SECURITY & PROTECTION LAYERS                    │
├────────────────────────────────────────────────────────────────────────┤
│ 1. Authentication: JWT signed with HS256, 7-day expiry, 32-byte secret │
│ 2. Password Security: Passwords hashed with bcrypt + auto salt         │
│ 3. Multi-Tenancy: Row-Level & Payload user_id scoping on all DB queries│
│ 4. Rate Limiting: Redis sliding-window limit (2 requests / 20 min)     │
│ 5. Input Validation: Strict Pydantic V2 schemas strip malicious payloads│
│ 6. Injection Defense: Parameterized SQL (SQLAlchemy) + tsquery parsing │
│ 7. Secret Management: All API keys stored in .env, never in Git        │
│ 8. CORS Protection: Explicit origin allowlists on FastAPI middleware   │
└────────────────────────────────────────────────────────────────────────┘
```

---

### Key Security Interview Questions & Answers

#### Q1: "How do you prevent SQL Injection and Prompt Injection?"
* **Answer:** 
  * **SQL Injection:** We use SQLAlchemy 2.0 with parameterized queries. No raw SQL strings are concatenated with user input.
  * **Prompt Injection:** User queries are never concatenated directly into system instructions. We pass retrieved documents and user queries into strictly formatted XML/markdown delimiter tags (e.g., `<context>{chunks}</context>`), and system prompts instruct the model to ignore any instructions found inside document chunks."

#### Q2: "How do you guarantee that User A cannot access User B's uploaded files?"
* **Answer:** "Every document, chunk, and vector is explicitly tagged with `user_id` upon creation. In PostgreSQL, all SELECT queries filter by `WHERE user_id = :current_user_id`. In Qdrant, every vector search includes a mandatory payload filter `FieldCondition(key='user_id', match=MatchValue(value=current_user_id))`. Because this filter is applied at the database layer, cross-tenant data access is impossible."

---

## 14. Testing Strategy

```
                       ┌─────────────────────────┐
                       │  LLM-as-a-Judge Eval    │  Quality / Faithfulness
                       ├─────────────────────────┤
                       │  End-to-End API Tests   │  FastAPI TestClient / HTTP
                       ├─────────────────────────┤
                       │  Integration Tests      │  DB, Qdrant, Celery Tasks
                       ├─────────────────────────┤
                       │  Unit Tests (pytest)    │  Parsers, Chunkers, Auth
                       └─────────────────────────┘
```

### Testing Breakdown:
1. **Unit Testing (`pytest`):** Tests individual functions in isolation: text cleaning regexes, chunk split lengths, password hashing, and token generation.
2. **Integration Testing:** Tests database migrations (Alembic), Qdrant vector upsert/search workflows, and Celery task execution.
3. **API Testing (`httpx` + `FastAPI TestClient`):** Simulates HTTP requests against `/auth/register`, `/auth/login`, and `/research/upload` to verify status codes and validation errors.
4. **Evaluation Harness (LLM-as-a-Judge):** Runs 20 automated test questions against the golden dataset to evaluate retrieval quality and hallucination rates.

---

## 15. Deployment and DevOps

```
                     ┌─────────────────────────────────────────┐
                     │          Vercel (Free Tier)             │
                     │          Next.js 16 Frontend            │
                     └────────────────────┬────────────────────┘
                                          │ HTTPS
                                          ▼
                     ┌─────────────────────────────────────────┐
                     │     Hugging Face Spaces / Docker        │
                     │    FastAPI Backend + Celery Worker      │
                     └──────┬─────────────┬─────────────┬──────┘
                            │             │             │
                            ▼             ▼             ▼
                     ┌──────────┐  ┌─────────────┐  ┌──────────┐
                     │   Neon   │  │   Upstash   │  │  Qdrant  │
                     │ Postgres │  │ Redis Cloud │  │  Cloud   │
                     └──────────┘  └─────────────┘  └──────────┘
```

### Deployment Configuration
* **Local Development:** One-command startup via `docker-compose up --build` which boots Postgres, Redis, Qdrant, Backend, Celery Worker, and Frontend.
* **Production Cloud Deployment:**
  * **Frontend:** Hosted on **Vercel** with automatic GitHub CI/CD deployments.
  * **Backend & Worker:** Containerized via Docker and deployed on **Hugging Face Spaces / Render**.
  * **Managed Cloud Databases:** **Neon Serverless PostgreSQL**, **Qdrant Cloud** (free tier cluster), and **Upstash Serverless Redis**.
* **Observability:** Optional **Langfuse** integration captures trace spans for token usage, latency, and LLM call histories.

---

## 16. Scalability Discussion

### Scaling from 1,000 to 1,000,000 Users

| Scale Stage | Key Bottlenecks | Engineering Solutions |
| :--- | :--- | :--- |
| **1,000 Users** | Single server resources, local memory limits. | Standard Docker Compose setup on a single VM. PostgreSQL and Qdrant run with default connection pools. |
| **10,000 Users** | Ingestion queue backlog, synchronous API thread blocking. | Separate Celery worker instances across multiple nodes. Use Redis cluster for task broker. Enable connection pooling via PgBouncer. |
| **100,000 Users** | Vector search query latency, relational DB read contention. | Shard Qdrant collections across multiple vector nodes. Introduce PostgreSQL Read Replicas. Implement Redis caching for frequent queries and embeddings. |
| **1,000,000 Users** | Global network latency, massive concurrent streaming load. | Deploy backend across multiple geographic regions with Anycast routing. Put Next.js assets behind Cloudflare CDN. Implement multi-cluster distributed Qdrant. |

---

### Interview Answer: "How would you scale this system to 1 million users?"
> "To scale Verity to 1 million users, I would address three key tiers:
> 1. **Stateless Compute Layer:** Run our FastAPI backend as horizontal microservices behind an Application Load Balancer (ALB) or Kubernetes (EKS), auto-scaling based on CPU and request latency.
> 2. **Distributed Ingestion Pipeline:** Scale Celery workers horizontally across spot instances with dedicated Redis queues for priority document processing.
> 3. **Database & Vector Sharding:** Implement PgBouncer for PostgreSQL connection pooling with read replicas for read-heavy operations. In Qdrant, shard vector collections by tenant hash across a distributed cluster, and use Redis to cache embeddings for popular queries to avoid redundant computation."

---

## 17. Impact of the Project

### Summary Metrics & KPIs

```
┌────────────────────────────────────────────────────────────────────────┐
│                        MEASURED PROJECT IMPACT                         │
├────────────────────────────────────────────────────────────────────────┤
│ • Context Recall:     0.32 ──► 0.45 (+41% Relative Improvement)       │
│ • Context Precision:  0.63 ──► 0.73 (+16% Relative Improvement)       │
│ • Faithfulness Score: 89% grounded factual accuracy (LLM Judge)        │
│ • Time-to-First-Token: < 600ms on streaming query endpoint             │
│ • Memory Optimization: 80% RAM reduction using FastEmbed ONNX          │
│ • Ingestion Speed:    10-page PDF parsed, cleaned & indexed in < 3s   │
└────────────────────────────────────────────────────────────────────────┘
```

### Business & User Impact
* **Time Savings:** Reduces literature review time for a 15-paper cluster from **8 hours to under 30 seconds**.
* **Zero Trust Hallucination Prevention:** Users can click and verify every statement against ground-truth source chunks.
* **Cost Efficiency:** Running FastEmbed ONNX locally eliminates expensive 3rd-party embedding API costs.

---

## 18. Future Enhancements

### Short-Term Improvements (1–3 Months)
* **True Semantic Chunking:** Implement dynamic cosine boundary chunking (splitting text when adjacent sentence similarity drops below a threshold) rather than fixed recursive token counts.
* **Full-Pipeline OpenTelemetry Tracing:** Expand Langfuse tracing to monitor retriever and reranker spans, not just generation.

### Medium-Term Improvements (3–6 Months)
* **GraphRAG Visual Knowledge Graph:** Expose an interactive 3D entity graph in the Next.js UI showing connections between papers, authors, and concepts.
* **Direct Audio/Video Ingestion:** Integrate Whisper models to transcribe and index YouTube research talks and podcasts directly into the knowledge base.

### Long-Term Vision (6–12 Months)
* **Autonomous Multi-Document Comparative Synthesis:** Enable a "Debate Agent" mode where two agents review conflicting papers, cross-examine methodologies, and generate a balanced consensus report.
* **On-Device Local LLM Support:** Support WebGPU / Ollama integration for 100% offline, private RAG execution.

---

## 19. Complete HR + Technical Interview Q&A

---

### Section A: 20 HR & Behavioral Questions (with Simple, Strong Answers)

#### 1. "Tell me about yourself."
> "I am a Full Stack / Software Engineer with strong experience in Python, FastAPI, React, Next.js, and modern AI architectures like RAG and multi-agent systems. I enjoy building end-to-end, high-performance web applications that solve real-world data problems. Recently, I built Verity, a production-grade Agentic RAG platform that eliminates hallucinations in academic research. I am looking for a challenging Software Engineer role where I can contribute to core product development and scalable backend systems."

#### 2. "What are your greatest strengths?"
> "My greatest strength is my ability to bridge frontend, backend, and distributed systems. When building features, I think about end-to-end user experience, API performance, database indexing, and system reliability. I am also a fast learner who loves diving deep into official documentation and technical papers."

#### 3. "What is an area of improvement you are working on?"
> "Because I love building and perfecting features, I sometimes spend extra time optimizing edge cases early. I have learned to practice strict agile prioritization—building a clean, working MVP first and using data and metrics to decide what to optimize next."

#### 4. "Why do you want to join our company?"
> "I admire your engineering culture, scale, and focus on high-quality software products. I want to work with talented engineers where I can tackle challenging scalability problems, write clean, robust code, and build features that impact millions of users."

#### 5. "Describe a difficult technical problem you solved."
> "In my RAG project, vector search alone was missing exact technical keywords and error codes. Instead of accepting this limitation, I researched hybrid search architectures, implemented PostgreSQL full-text search alongside Qdrant vector search, fused them with Reciprocal Rank Fusion, and added an ONNX cross-encoder. This boosted context recall by 41%."

#### 6. "How do you handle deadlines and pressure?"
> "I break down large projects into small, well-defined milestones and prioritize critical path features first. I maintain clear documentation and communicate early if blockers arise so there are no last-minute surprises."

#### 7. "Tell me about a time you made a mistake. How did you fix it?"
> "Early in the project, I ran heavy PDF text extraction directly inside an async FastAPI route, which blocked the event loop and caused other requests to time out. When I noticed the issue in logs, I immediately refactored the architecture to offload all heavy ingestion tasks to asynchronous Celery workers backed by Redis."

#### 8. "How do you keep your technical skills up to date?"
> "I read engineering blogs from companies like Google, Uber, and Netflix, study official documentation, read ArXiv research papers on AI systems, and build hands-on side projects to test new technologies."

#### 9. "Do you prefer working on frontend or backend?"
> "I am comfortable across the full stack. I enjoy the logic, data modeling, and performance optimization of backend systems, but I also value building polished, intuitive user interfaces with Next.js and TypeScript."

#### 10. "Where do you see yourself in 3 to 5 years?"
> "In 3 to 5 years, I see myself as a Senior Software Engineer leading core system architecture, mentoring junior engineers, and taking ownership of large-scale distributed systems."

#### 11. "How do you handle receiving critical feedback on a pull request?"
> "I view code reviews as an opportunity to learn. I appreciate constructive feedback because it improves code quality and security. If a reviewer suggests a better approach, I test it, understand the reasoning, and apply the learning."

#### 12. "How do you prioritize features when you have limited time?"
> "I use an impact vs. effort framework. I identify the core functionality that provides 80% of the user value—such as search accuracy and streaming speed—and prioritize those before secondary nice-to-have features."

#### 13. "Have you ever worked with ambiguous requirements? What did you do?"
> "Yes. When requirements are ambiguous, I ask clarifying questions, create simple architecture diagrams, build a minimal prototype, and align with stakeholders before writing the full implementation."

#### 14. "What does clean code mean to you?"
> "Clean code is readable, modular, self-documenting, and well-tested. It follows standard design principles like DRY and SOLID, has clear variable names, and can be easily maintained by any engineer on the team."

#### 15. "How do you approach learning a completely new technology?"
> "I start by reading the official documentation and understanding the core mental model. Then I build a small proof-of-concept project to test edge cases, error handling, and performance before using it in production."

#### 16. "Tell me about a time you had to explain a complex technical concept to a non-technical person."
> "When explaining RAG to non-technical users, I used the analogy of an open-book exam: standard AI tries to answer from memory and sometimes guesses wrong, while RAG lets the AI look up the exact textbook page before writing its answer."

#### 17. "What motivates you as an engineer?"
> "Building software that works reliably, solves a real frustration for people, and performs with speed and elegance. Solving difficult technical puzzles gives me huge satisfaction."

#### 18. "How do you ensure your code is secure?"
> "I follow security best practices: never trusting client input (using Pydantic schemas), using parameterized SQL queries, implementing rate limiting, hashing passwords with bcrypt, and keeping API secrets in secure environment variables."

#### 19. "What are your salary expectations?"
> "I am looking for a competitive package aligned with industry standards for this role and location. I am very enthusiastic about the opportunity to contribute to your team and am open to discussing fair compensation."

#### 20. "Do you have any questions for us?"
> "Yes! What are the biggest technical challenges your engineering team is currently solving? And what does the typical engineering workflow look like from feature design to production deployment?"

---

### Section B: 30 Core Technical Questions (Simple & Direct)

#### 1. "What is an asynchronous event loop in Python?"
* **Answer:** "An event loop is a continuous loop that monitors and executes asynchronous tasks. When an I/O task (like a database query or network request) starts, the event loop pauses that task and runs other ready tasks, enabling high concurrency on a single thread."

#### 2. "What is the difference between synchronous and asynchronous code?"
* **Answer:** "Synchronous code executes line-by-line, blocking execution until the current operation completes. Asynchronous code allows long-running I/O operations to run in the background, freeing the thread to handle other tasks in the meantime."

#### 3. "What is the difference between SQL and NoSQL databases?"
* **Answer:** "SQL databases (like PostgreSQL) are relational, use structured schemas with tables, support ACID transactions, and excel at complex relational queries. NoSQL databases (like MongoDB or Redis) are non-relational, offer flexible schemas, and are optimized for specific access patterns like key-value lookup or document storage."

#### 4. "What is an index in a database, and how does it work?"
* **Answer:** "An index is a data structure (commonly a B-Tree or GIN) that allows the database to find rows in $O(\log N)$ time instead of scanning every row sequentially ($O(N)$ full table scan)."

#### 5. "What is a GIN index in PostgreSQL?"
* **Answer:** "A GIN (Generalized Inverted Index) maps words or elements to the rows where they appear. It is ideal for full-text search (`tsvector`) and JSONB arrays because it allows fast lookup of documents containing specific words."

#### 6. "What is the difference between GET and POST HTTP methods?"
* **Answer:** "GET requests retrieve data from the server and should be idempotent with no side effects. POST requests send data to the server in the request body to create or process resources."

#### 7. "What is JWT (JSON Web Token) and how does it work?"
* **Answer:** "A JWT is a stateless, digitally signed token containing three parts: Header, Payload, and Signature. The server verifies the signature using a secret key, confirming the user's identity without querying a session database on every request."

#### 8. "Why should you hash passwords with bcrypt instead of MD5 or SHA256?"
* **Answer:** "MD5 and SHA-256 are fast general-purpose hash functions, making them vulnerable to brute-force attacks. `bcrypt` is intentionally slow, computationally expensive, and includes a built-in cryptographic salt to prevent rainbow table attacks."

#### 9. "What is CORS (Cross-Origin Resource Sharing)?"
* **Answer:** "CORS is a browser security mechanism that restricts web pages from making API requests to a different domain, protocol, or port unless the server explicitly sends `Access-Control-Allow-Origin` headers allowing it."

#### 10. "What is the difference between Server-Sent Events (SSE) and WebSockets?"
* **Answer:** "SSE is unidirectional (server-to-client) over standard HTTP, making it ideal for streaming text. WebSockets are bidirectional (two-way) over a persistent TCP connection, suitable for multiplayer games or live collaborative editing."

#### 11. "What is a Celery task queue?"
* **Answer:** "Celery is an asynchronous task queue in Python. It allows web applications to offload time-consuming tasks (like PDF parsing or email sending) to background worker processes via a message broker like Redis."

#### 12. "What is Redis and why is it so fast?"
* **Answer:** "Redis is an in-memory data store. It is extremely fast (sub-millisecond latency) because all data resides in RAM rather than on disk, and it uses an optimized single-threaded event loop."

#### 13. "What is Docker and why is it used?"
* **Answer:** "Docker is a containerization platform that packages an application and all its dependencies into an isolated container image, ensuring the software runs identically across local development, staging, and production."

#### 14. "What is the difference between a Docker image and a container?"
* **Answer:** "A Docker image is a static, read-only blueprint/template containing the application code and environment. A container is a running, live instance of an image."

#### 15. "What is Pydantic in FastAPI?"
* **Answer:** "Pydantic is a data validation library that uses Python type hints. It automatically parses, type-checks, and validates incoming JSON request bodies, returning clear 422 validation errors if the data is malformed."

#### 16. "What is an ORM (Object-Relational Mapper)?"
* **Answer:** "An ORM (like SQLAlchemy) allows developers to query and manipulate database tables using object-oriented Python code instead of writing raw SQL strings."

#### 17. "What is a connection pool in database engineering?"
* **Answer:** "A connection pool maintains a cache of open database connections. Instead of opening and closing a new TCP connection on every HTTP request (which is slow), the app borrows an existing connection and returns it when done."

#### 18. "What is Rate Limiting and why is it necessary?"
* **Answer:** "Rate limiting restricts the number of API requests a client can make in a given timeframe, protecting backend services from denial-of-service (DoS) attacks, brute-force login attempts, and excessive API costs."

#### 19. "What is the difference between React state and props?"
* **Answer:** "Props are read-only inputs passed from a parent component to a child. State is mutable internal data managed inside the component that triggers a UI re-render when updated."

#### 20. "What is the virtual DOM in React?"
* **Answer:** "The Virtual DOM is an in-memory lightweight representation of the real DOM. When state changes, React compares the new Virtual DOM with the previous snapshot (diffing) and updates only the changed elements in the real DOM (reconciliation)."

#### 21. "What are React Hooks? Name three common hooks."
* **Answer:** "Hooks are functions that let you use state and lifecycle features in functional React components. Examples: `useState` (manages local state), `useEffect` (manages side effects like API calls), and `useRef` (holds mutable references without causing re-renders)."

#### 22. "What is TypeScript and what are its advantages?"
* **Answer:** "TypeScript is a statically typed superset of JavaScript. It provides compile-time type checking, autocomplete, and interface definitions, catching bugs before code runs in production."

#### 23. "What is Next.js App Router?"
* **Answer:** "Next.js App Router is a file-system-based routing architecture that supports React Server Components, streaming with Suspense, nested layouts, and simplified server-side data fetching."

#### 24. "What is the difference between Client Components and Server Components in Next.js?"
* **Answer:** "Server Components render exclusively on the server, producing zero JavaScript bundle size on the client. Client Components (marked with `'use client'`) run on both server and browser to support interactive hooks like `useState`, `useEffect`, and browser event listeners."

#### 25. "What is ACID in database systems?"
* **Answer:** "ACID stands for Atomicity (all-or-nothing transactions), Consistency (valid state transitions), Isolation (concurrent transactions don't interfere), and Durability (committed changes survive system crashes)."

#### 26. "What is the difference between a Process and a Thread?"
* **Answer:** "A process is an independent executing program with its own dedicated memory space. A thread is a lightweight unit of execution within a process that shares memory with other threads in the same process."

#### 27. "What is the Python GIL (Global Interpreter Lock)?"
* **Answer:** "The GIL is a mutex in CPython that prevents multiple native threads from executing Python bytecode simultaneously, meaning CPU-bound Python threads cannot run in true parallel on multiple cores within a single process."

#### 28. "How do you achieve true parallelism in Python for CPU-heavy tasks?"
* **Answer:** "By using multiprocessing (e.g., Python's `multiprocessing` module or Celery worker processes) which spawns separate OS processes, each with its own Python interpreter and GIL."

#### 29. "What is a 401 Unauthorized vs. a 403 Forbidden HTTP status code?"
* **Answer:** "401 Unauthorized means the client has not provided valid authentication credentials. 403 Forbidden means the client is authenticated, but does not have permission to access the requested resource."

#### 30. "What is a Reverse Proxy (like NGINX)?"
* **Answer:** "A reverse proxy sits in front of backend web servers, handling SSL termination, load balancing, caching, and routing client requests securely to internal backend services."

---

### Section C: 20 Architecture & System Design Questions

#### 1. "Explain the high-level architecture of your project."
* **Answer:** "Verity uses a 3-tier architecture: A Next.js 16 frontend for UI and SSE streaming, a FastAPI backend for API routing and LangGraph agent orchestration, and a storage tier consisting of PostgreSQL for relational data and full-text search, Qdrant for vector search, and Redis for Celery background tasks."

#### 2. "How does the dual-path query architecture work?"
* **Answer:** "We separate user interactions into a low-latency path (`/research/query/stream`) that uses fast hybrid retrieval and immediate SSE token streaming for live chat, and a full agentic path (`/research/query`) that runs LangGraph nodes for query planning, grading, contradiction detection, and citation verification for deep research."

#### 3. "What is Hybrid Search and why is it better than dense-only search?"
* **Answer:** "Hybrid search combines dense vector retrieval (semantic concepts via Qdrant) and sparse keyword retrieval (exact tokens via PostgreSQL FTS). Combining both ensures we don't miss exact keywords or domain-specific acronyms while still matching conceptual meaning."

#### 4. "How does Reciprocal Rank Fusion (RRF) work?"
* **Answer:** "RRF is an algorithm that combines rankings from multiple search systems without needing score normalization. It calculates a score for each document as:
  $$\text{Score}(d) = \sum_{m \in \text{models}} \frac{1}{k + \text{rank}_m(d)}$$
  where $k=60$ is a constant. Documents ranked high in both systems float to the top."

#### 5. "What is a Cross-Encoder Reranker and where does it fit in the pipeline?"
* **Answer:** "A bi-encoder creates embeddings for query and document independently (fast, but misses cross-interactions). A cross-encoder passes the query and document together through transformer layers to compute exact joint relevance. We use it in Stage 3 to rescore the top 30 RRF candidates down to the top 8 highest-quality chunks."

#### 6. "What is HyDE (Hypothetical Document Embeddings)?"
* **Answer:** "HyDE asks an LLM to generate a hypothetical answer to the user's question first. Then, it embeds that hypothetical answer to search the vector database. Because a hypothetical answer looks more like a real document chunk than a short query does, it yields superior vector similarity matches."

#### 7. "How does your system handle multi-tenant data isolation?"
* **Answer:** "Every document chunk and vector is tagged with the user's `user_id`. In PostgreSQL, queries are scoped with `WHERE user_id = :id`. In Qdrant, we use payload filter conditions matching `user_id`, ensuring no user can ever search or retrieve another user's documents."

#### 8. "How does the background Celery ingestion pipeline operate?"
* **Answer:** "When a file is uploaded, the FastAPI route creates a database record, returns `202 Accepted`, and dispatches a Celery task. The Celery worker parses the file, extracts images, cleans text, computes ONNX embeddings, and indexes vectors in Qdrant and PostgreSQL asynchronously."

#### 9. "What is an Adaptive Knowledge Base (Self-Improving RAG)?"
* **Answer:** "If the hybrid retriever finds that the best cosine similarity score is below 0.45, it triggers a Web Search Agent. The web results are used to answer the user immediately, and a background Celery task cleans, chunks, and embeds the web content into the user's private database for future queries."

#### 10. "Why did you use LangGraph instead of standard LangChain chains?"
* **Answer:** "LangChain chains are strictly linear directed acyclic graphs (DAGs). LangGraph supports cyclic state graphs with conditional branches, loops, and custom state reducers, which are essential for agentic loops (e.g., retrying search if grading fails)."

#### 11. "How do you detect contradictions between research papers?"
* **Answer:** "Our LangGraph pipeline includes a Contradiction Detector node. It prompts the LLM to inspect retrieved chunks for mutually exclusive factual claims, tagging differences and explicitly alerting the user if authors disagree."

#### 12. "How do you prevent connection pool exhaustion in FastAPI?"
* **Answer:** "We use SQLAlchemy async sessions managed via async context managers (`async with AsyncSession() as session:`). This ensures every connection is returned to the pool immediately after execution, even if an exception occurs."

#### 13. "What happens if a user disconnects during an SSE stream?"
* **Answer:** "The streaming generator checks `await request.is_disconnected()` on every token iteration. If true, it breaks out of the loop and exits cleanly, closing downstream connections."

#### 14. "How do you handle large file uploads without consuming all server RAM?"
* **Answer:** "We stream the incoming file in 1MB chunks from `UploadFile.file` directly to disk in a temporary directory, rather than reading the entire file into memory at once."

#### 15. "How is full-text search integrated with relational tables in PostgreSQL?"
* **Answer:** "We store a `tsvector` column on the `document_chunks` table that automatically indexes chunk text. A GIN index on this column enables rapid keyword search using `to_tsquery()`."

#### 16. "What is an ONNX Runtime and why use it for embeddings?"
* **Answer:** "ONNX (Open Neural Network Exchange) is an open format for ML models. The ONNX Runtime is a high-performance C++ inference engine that runs quantized models with minimal RAM and without needing heavy frameworks like PyTorch or TensorFlow."

#### 17. "How do you prevent duplicate file uploads?"
* **Answer:** "We compute a SHA-256 hash of the uploaded file contents. Before parsing, we check if a document with that hash and `user_id` exists in PostgreSQL. If found, we skip ingestion immediately."

#### 18. "How does the LLM-as-a-judge evaluation harness work?"
* **Answer:** "It runs a benchmark script over 20 curated gold-standard Q/A pairs. An independent LLM judges the pipeline's answers on four standard RAG metrics: Faithfulness, Answer Relevancy, Context Recall, and Context Precision."

#### 19. "How do you configure CORS securely in FastAPI?"
* **Answer:** "We use FastAPI's `CORSMiddleware` and specify an explicit list of allowed origins from environment variables (e.g., `http://localhost:3000`), explicitly rejecting wildcards (`*`) when credentials/cookies are involved."

#### 20. "How would you scale this architecture horizontally?"
* **Answer:** "The FastAPI backend is completely stateless; we can scale it horizontally across multiple container instances behind a load balancer. We scale Celery workers independently, add PgBouncer for DB connection pooling, and use a distributed Qdrant cluster."

---

### Section D: 20 Project Deep-Dive Questions (Verity Specific)

#### 1. "What exact embedding model does Verity use and why?"
* **Answer:** "Verity uses `BAAI/bge-small-en-v1.5`, producing 384-dimensional embeddings. We chose it because it is lightweight, achieves top ranking on the MTEB leaderboard for its size class, and runs efficiently in-memory via FastEmbed ONNX."

#### 2. "What cross-encoder model is used for reranking?"
* **Answer:** "We use `cross-encoder/ms-marco-MiniLM-L-6-v2` compiled for ONNX. It rescores query-document pairs on a continuous relevance scale without PyTorch dependencies."

#### 3. "What LLMs are integrated into Verity?"
* **Answer:** "We use **Llama 3.3 70B Instruct** via OpenRouter for high-quality reasoning and generation, **Gemma 3 4B** for fast planning and grading, and **Llama 4 Scout** for multi-modal figure and diagram descriptions."

#### 4. "How are PDF figures and charts processed?"
* **Answer:** "During ingestion, PyMuPDF extracts embedded images and figures from the PDF. The images are sent to a vision LLM to generate descriptive textual captions, which are stored alongside the text chunks so charts become searchable."

#### 5. "What chunking strategies are implemented in Verity?"
* **Answer:** "We implemented structural chunkers: Recursive Character chunking for general text and PDFs (512 tokens with 50 overlap), Markdown chunking split by header tags, and Heading chunking for DOCX files."

#### 6. "How does the Citation Verifier node work in code?"
* **Answer:** "It extracts all regex patterns matching `\[(\d+)\]` from the generated answer. It verifies that index $N$ exists in the retrieved context chunks and checks that the referenced chunk contains semantic support for the sentence."

#### 7. "What is RAPTOR and how does Verity implement it?"
* **Answer:** "RAPTOR recursively clusters text chunks using Gaussian Mixture Models on embeddings, generates summaries of each cluster with an LLM, and embeds those summaries. This allows the system to answer high-level questions like *'What are the overarching themes of this paper?'*"

#### 8. "What rate limiting strategy is used on the streaming route?"
* **Answer:** "A Redis sliding-window counter tracking client IP / User ID. It permits 2 requests per 20 minutes on the free demo tier, returning an HTTP 429 status code with a `Retry-After` header when exceeded."

#### 9. "What happens when a document contains noisy academic metadata?"
* **Answer:** "Our cleaning pipeline applies regex filters that strip out author email headers, arXiv submission stamps, DOI URLs, journal copyright notices, and page footers before passing text to the chunker."

#### 10. "How does the Grader Agent decide whether to search the web?"
* **Answer:** "The Grader Agent passes the query and top retrieved chunks to a fast LLM prompt that outputs a binary `yes/no` relevance grade. If the grade is `no` or if no chunks were retrieved, the LangGraph routing edge transitions to the Web Search node."

#### 11. "Which web search providers are supported in the fallback agent?"
* **Answer:** "We support **Tavily Search API** (optimized for AI agents) as primary, with an automatic fallback to **DuckDuckGo Search** if no Tavily API key is provided."

#### 12. "How do you store and manage conversation history in memory?"
* **Answer:** "We store past message turns in the PostgreSQL `chat_messages` table linked to a `session_id`. When querying, the last $K$ turns are loaded to provide conversational context to the prompt."

#### 13. "What metrics did the evaluation harness report for the cross-encoder reranker?"
* **Answer:** "On a 13-paper RAG corpus evaluated against 20 curated Q/A pairs, adding the cross-encoder reranker increased **Context Recall from 0.32 to 0.45 (+41%)** and **Context Precision from 0.63 to 0.73 (+16%)**."

#### 14. "Why did Answer Relevancy remain at 0.97 before and after reranking?"
* **Answer:** "Answer Relevancy was already at the ceiling (0.97) because the LLM is good at generating fluent, relevant-sounding answers. The reranker's true impact was on **retrieval-side grounding metrics** (recall and precision), ensuring the answer was based on the right factual chunks."

#### 15. "How is JWT authentication implemented in FastAPI?"
* **Answer:** "We use Python's `pyjwt` library. When a user logs in, we sign a token containing `sub: user_id` and `exp` timestamp using a 256-bit secret key. Protected endpoints use `Depends(get_current_user)` to extract and validate the token from the `Authorization: Bearer <token>` header."

#### 16. "What UI library is used for chat components in Next.js?"
* **Answer:** "We use **shadcn/ui** components styled with Tailwind CSS, supporting auto-scrolling chat lists, Markdown formatting with code syntax highlighting, and expandable source citation sheets."

#### 17. "How are database schema migrations handled?"
* **Answer:** "We use **Alembic**. Database migrations are version-controlled Python migration scripts in `/migrations`. Running `alembic upgrade head` applies schema changes automatically during container startup."

#### 18. "What is the role of `scripts/arxiv_ingest.py`?"
* **Answer:** "It is a utility CLI script that queries the ArXiv API for a given search query (e.g. `'retrieval augmented generation'`), downloads the top $N$ papers, and runs them through our ingestion pipeline automatically."

#### 19. "How does the system handle multi-modal PDF inputs?"
* **Answer:** "PyMuPDF extracts embedded raster images from PDF pages. We pass images to OpenRouter's vision endpoint (Llama 4 Scout), receive a rich textual summary of the chart/diagram, and embed that summary alongside adjacent text."

#### 20. "What is the single most important architectural takeaway from this project?"
* **Answer:** "That high-quality RAG is a software engineering problem, not just a prompt engineering problem. True accuracy comes from clean data pipelines, hybrid retrieval, cross-encoder reranking, and deterministic verification agents."

---

## 20. Final Interview Script ("Explain Your Project")

---

### 30-Second Version (Clear & Punchy)
> "I built **Verity**, an Agentic RAG platform that allows users to upload complex research papers and get verified, hallucination-free answers. 
> 
> It is built with **Next.js 16** on the frontend with Server-Sent Events for real-time streaming, and **FastAPI, PostgreSQL, and Qdrant** on the backend. 
> 
> It uses **Hybrid Retrieval** combining semantic vector search and keyword full-text search, rescored by a local **Cross-Encoder Reranker**, and verified by **LangGraph agents** that check citations and detect contradictions between sources."

---

### 1-Minute Version (Structured & Professional)
> "My project is **Verity**, a full-stack Agentic RAG platform designed to eliminate hallucinations when analyzing technical and academic documents.
> 
> The architecture consists of a **Next.js 16** frontend with real-time SSE streaming, and a **FastAPI** backend in Python 3.12. 
> 
> When a document is uploaded, it is cleaned, chunked, and indexed into **Qdrant** for dense vector search and **PostgreSQL** with GIN indexes for sparse keyword search. 
> 
> During a query, we execute hybrid retrieval fused by **Reciprocal Rank Fusion (RRF)** and rescored by a local **ONNX Cross-Encoder**. To ensure high accuracy, we orchestrate **LangGraph agents** that plan queries, verify bracketed citations, detect conflicting claims between papers, and automatically fall back to web search if confidence is low. 
> 
> The system includes JWT auth with per-user tenant isolation, Celery background workers, and an automated evaluation harness that demonstrated a **41% increase in context recall**."

---

### 3-Minute Version (Comprehensive Full-Stack Walkthrough)
> "Let me share **Verity**, a production-grade Agentic RAG platform I designed and built from scratch.
> 
> **The Problem:** Standard RAG pipelines suffer from major failure modes: dense vector search misses exact technical keywords, LLMs hallucinate citations, and systems fail when two research papers contradict each other.
> 
> **Frontend Architecture:**
> Built with **Next.js 16**, **TypeScript**, and **Tailwind CSS** with **shadcn/ui**. It provides a responsive chat interface that consumes token-by-token streaming via **Server-Sent Events (SSE)**, renders LaTeX math and Markdown code blocks, and allows direct in-chat drag-and-drop document uploads.
> 
> **Backend & Retrieval Architecture:**
> The backend is built with **FastAPI** and **Python 3.12**. 
> To solve the retrieval problem, I built a 3-stage **Hybrid Retrieval Pipeline**:
> 1. Stage 1 concurrently queries **Qdrant** for dense semantic vectors (using 384-dimensional `bge-small-en-v1.5` embeddings) and **PostgreSQL** for sparse BM25 keyword matching using `tsvector` and GIN indexing.
> 2. Stage 2 combines both rankings using **Reciprocal Rank Fusion (RRF)** with constant $k=60$.
> 3. Stage 3 rescores the top 30 candidates using an **ONNX Cross-Encoder** (`ms-marco-MiniLM`), running locally with zero PyTorch overhead.
> 
> **Agentic Orchestration & Verification:**
> We use **LangGraph** to model multi-agent workflows. A **Planner Agent** expands queries using Hypothetical Document Embeddings (HyDE). A **Grader Agent** checks chunk relevance; if confidence is low, it triggers a **Web Search Agent** that synthesizes the answer and enqueues a background **Celery worker** to ingest the web data into the user's private database. A **Contradiction Detector** identifies conflicting facts between papers, and a **Citation Verifier** ensures every bracketed citation `[1][2]` matches verified source text.
> 
> **Security & DevOps:**
> We implement stateless **JWT authentication** with bcrypt password hashing and enforce strict multi-tenant isolation at the database layer. Heavy file ingestion runs asynchronously on **Celery with Redis**.
> 
> Finally, I built an **LLM-as-a-judge evaluation harness** that proved our reranker improved **Context Recall by 41%** and **Context Precision by 16%** over baseline RAG."

---

### 5-Minute Deep Dive (Senior Engineering Level)
> "I would like to present **Verity**, a production-grade Agentic Retrieval-Augmented Generation (RAG) platform. I built Verity to move beyond toy AI tutorials and engineer a full-stack, enterprise-ready system that guarantees factual correctness and verified citations across complex document libraries.
> 
> Let me walk you through the four core layers of the system:
> 
> ### 1. Document Ingestion & Processing Pipeline
> When a user uploads a PDF, Word document, or URL, the request is handled asynchronously:
> * **Idempotency:** We compute a SHA-256 hash to prevent duplicate ingestion and wasted storage.
> * **Multi-Modal Extraction:** PyMuPDF extracts text and figures. Figures are passed to vision LLMs to produce text summaries of charts and architectures.
> * **Text Cleaning:** A specialized regex pipeline strips academic noise: email headers, DOIs, page numbers, and copyright stamps.
> * **Chunking & Embedding:** Documents are chunked into 512-token segments and embedded using `bge-small-en-v1.5` via **FastEmbed ONNX**. Vectors are upserted to **Qdrant** with user-scoped payloads, while chunk text and generated `tsvector` data are stored in **PostgreSQL**.
> 
> ### 2. Hybrid Retrieval & Cross-Encoder Reranking Engine
> Most naive RAG apps fail on exact keywords like error codes or acronyms. To fix this, I engineered a 3-stage retrieval pipeline:
> * **Stage 1 (Candidate Fetch):** Concurrently retrieves Top 20 dense cosine vectors from Qdrant and Top 20 sparse keyword matches from PostgreSQL FTS.
> * **Stage 2 (Rank Fusion):** Fuses both candidate lists using Reciprocal Rank Fusion ($k=60$) into Top 30 candidates.
> * **Stage 3 (Cross-Encoder Rescoring):** Passes query-document pairs through a cross-encoder model in ONNX Runtime, computing joint semantic relevance to select the Top 8 chunks.
> 
> ### 3. LangGraph Multi-Agent Architecture
> To eliminate hallucinations, we orchestrate multiple specialized agents:
> * **Planner & HyDE:** Generates hypothetical document embeddings to expand complex queries.
> * **Grader Node:** Evaluates chunk relevance. If cosine similarity is below 0.45, it branches to a **Web Search Agent** (Tavily/DuckDuckGo), feeding live data to the user while dispatching a background Celery task to auto-ingest the web content.
> * **Contradiction Detector:** Flags opposing claims across different papers.
> * **Generator & Citation Verifier:** Streams answers with inline citations `[1][2]` and deterministically verifies references against the source payload before completion.
> 
> ### 4. Dual-Path Design, Security & Evaluation
> * **Dual-Path Latency Optimization:** For live chat, the `/research/query/stream` endpoint runs fast hybrid retrieval and streams tokens over Server-Sent Events (SSE) in under 600ms TTFT. The full `/research/query` endpoint executes the entire LangGraph verification graph for deep research.
> * **Multi-Tenancy & Security:** Stateless JWT authentication (`HS256`), bcrypt password hashing, Redis sliding-window rate limiting, and row/payload level `user_id` tenant scoping.
> * **Quantitative Evaluation:** Built an automated LLM-as-a-judge harness. Across a 13-paper corpus with 20 golden Q/A pairs, our cross-encoder reranker delivered a **+41% gain in Context Recall (0.32 to 0.45)** and a **+16% gain in Context Precision (0.63 to 0.73)** with an **89% faithfulness score**.
> 
> This project gave me deep hands-on expertise in distributed asynchronous systems, vector databases, multi-agent state machines, and high-performance full-stack web architecture."
