# 🧠 PaperMind — Multi-Agent AI Research Assistant
 
> A production-grade multi-agent system that autonomously searches 7 academic databases in parallel, synthesizes findings across papers, and delivers structured research reports with citation grounding, contradiction detection, and confidence scoring — streamed token by token in real time.
  
[![Python](https://img.shields.io/badge/Python-3.11-3776ab?logo=python&logoColor=white)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![LangGraph](https://img.shields.io/badge/LangGraph-1.1-6f42c1)](https://langchain-ai.github.io/langgraph)
[![React](https://img.shields.io/badge/React-18-61dafb?logo=react&logoColor=white)](https://react.dev)
[![Supabase](https://img.shields.io/badge/Supabase-Auth%20%2B%20pgvector-3ecf8e?logo=supabase&logoColor=white)](https://supabase.com)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)
 
---
 
## Demo:
https://drive.google.com/file/d/1N1kYi-Y_DtZqZEE0K56GQQRg1DAN9FZW/view?usp=sharing
 
> ▶ Click to watch — multi-agent research pipeline in action
 
---
 
## What it does
 
PaperMind deploys a team of 4 specialist AI agents orchestrated by a LangGraph supervisor:
 
1. **Search Agent** — queries Arxiv, OpenAlex, EuropePMC, Crossref, PubMed, Semantic Scholar, and CORE in parallel using `ThreadPoolExecutor`; deduplicates across sources; gracefully skips rate-limited APIs (429 handling)
2. **RAG Agent** — semantically retrieves relevant chunks from user-uploaded PDFs using Supabase pgvector + Gemini embeddings; cosine similarity threshold (0.45) filters out off-topic chunks; per-user isolation ensures no cross-contamination
3. **Summarizer Agent** — extracts structured citation-ready data from each paper with exact metrics, methodology tags, and relevance scoring via a single batched LLM call
4. **Synthesizer Agent** — produces a cross-paper synthesis with methodology distribution, numbered inline citations, contradiction detection, confidence scoring, and 5 contextual follow-up questions — streamed token by token in real time
 
Every factual claim is grounded with inline citations `[1][3]`. The report appears word-by-word as the LLM generates it.
 
---
 
## Architecture
 
```
User Query
    │
    ▼
FastAPI + WebSocket (Render)
    │
    ▼
LangGraph Supervisor ← deterministic routing via completed_steps
    │
    ├── Search Agent ──→ 7 APIs in parallel (ThreadPoolExecutor)
    │                    Arxiv · OpenAlex · EuropePMC · Crossref
    │                    PubMed · Semantic Scholar · CORE
    │
    ├── RAG Agent ──────→ Supabase pgvector (cosine similarity)
    │                    Gemini embeddings · per-user isolation
    │                    cosine threshold 0.45
    │
    ├── Summarizer ─────→ Batched LLM call · structured extraction
    │                    citation tagging · exact metrics
    │
    └── Synthesizer ────→ Cross-paper synthesis · token streaming
                         asyncio.to_thread + ContextVar injection
                         Follow-up questions (conditional LLM call)
                         Saves session to Supabase after completion
    │
    ▼
Supabase (Auth + pgvector + research_sessions)
React + Vite (Vercel)
```
 
---
 
## Tech Stack
 
| Layer | Technology | Why |
|---|---|---|
| Agent orchestration | LangGraph 1.1 | Explicit state graph, deterministic supervisor routing via `completed_steps` |
| Backend | FastAPI + WebSocket | Async, streaming-first, production-grade |
| Auth | Supabase Auth | Google OAuth + email/password, ES256 JWT via JWKS |
| Vector store | Supabase pgvector | Vectors colocated with user data in Postgres — transactional consistency, SQL-level filtering |
| Embeddings | Gemini gemini-embedding-001 | Free API, 3072-dim, zero local memory |
| LLM | Groq llama-3.3-70b | Free tier, <5s latency; auto-fallback to Gemini |
| PDF parsing | PyMuPDF | Fast, handles academic PDFs with text extraction |
| Academic APIs | Arxiv, OpenAlex, EuropePMC, Crossref, PubMed, S2, CORE | All free, no key required for most |
| Frontend | React + Vite | Token streaming, dark/light theme, history dropdown |
| Logging | python-json-logger | Structured JSON, per-request trace IDs via ContextVar |
| Deployment | Render + Vercel | Free tier, auto-deploy from GitHub |
 
**Total monthly cost: $0**
 
---
 
## Key Engineering Decisions
 
### Why LangGraph over CrewAI?
LangGraph exposes the full graph — nodes, edges, state schema, routing logic. The supervisor uses **deterministic rule-based routing** via a `completed_steps` list instead of LLM-based routing, eliminating hallucination in agent orchestration. CrewAI abstracts this away, which is fine for demos but makes production debugging impossible.
 
### Why `asyncio.to_thread` for LLM calls?
LangChain's sync `llm.invoke()` blocks the event loop. Every agent node offloads to a thread pool via `asyncio.to_thread`. The synthesizer uses `llm.stream()` inside the thread and emits tokens immediately via `emit_sync`, preventing Render's 90-second WebSocket idle timeout during long synthesis.
 
### Why ContextVar for stream injection?
The `StreamManager` needs to reach the synthesizer node, but LangGraph state is typed and shouldn't carry non-serialisable objects. `ContextVar` provides per-request context that propagates through `async` boundaries and thread pool workers without polluting the state schema.
 
### Why pgvector over ChromaDB?
ChromaDB + sentence-transformers loads ~400MB locally — exceeds Render's 512MB free tier. More importantly: with ChromaDB, user data lives in two separate systems (Supabase for auth, ChromaDB for vectors) that can go out of sync. pgvector colocates vectors with user records in Postgres, enabling a single `ON DELETE CASCADE` constraint to clean up all user data atomically.
 
### Why cosine distance threshold in RAG?
Without a threshold, pgvector always returns top-n chunks regardless of semantic similarity. A threshold of 0.45 means off-topic queries return zero chunks, preventing cross-domain contamination when multiple PDFs from different fields are uploaded.
 
### Why ES256 JWT verification via JWKS?
Modern Supabase projects issue asymmetric ES256 tokens. The backend fetches the public key from Supabase's JWKS endpoint (`/auth/v1/.well-known/jwks.json`), cached via `@lru_cache`. Zero shared secrets in production, native support for Google OAuth tokens.
 
### Why ContextVar for trace IDs?
Structured logging needs the same request ID across async boundaries and thread pool workers. `ContextVar` propagates the trace ID from the WebSocket handler through every agent node — including those running in `asyncio.to_thread` — without passing it as a function parameter.
 
---
 
## Features
 
- **Multi-agent pipeline** — 4 specialist agents with deterministic LangGraph supervisor
- **Parallel academic search** — 7 APIs simultaneously, 429 handling, cross-source deduplication
- **Per-user RAG** — each user's uploaded PDFs are isolated by `user_id` in pgvector
- **Real-time streaming** — report appears token by token via WebSocket
- **Query history** — sessions saved to Supabase, restored on click from search bar dropdown
- **Follow-up questions** — 5 contextual questions generated after report; toggle in UI
- **Multi-provider LLM fallback** — Groq → Gemini automatic failover, zero code change
- **Google OAuth + email auth** — Supabase Auth with ES256 JWT verification
- **Dark/light theme** — persisted in localStorage
- **Structured JSON logging** — per-request trace IDs via ContextVar
 
---
 
## Project Structure
 
```
papermind/
├── main.py                       # FastAPI app + RequestIdMiddleware
├── core/
│   ├── config.py                 # Pydantic settings from .env
│   ├── auth.py                   # ES256/HS256 JWT via JWKS + @lru_cache
│   ├── database.py               # Supabase client + session history CRUD
│   ├── llm.py                    # Multi-provider factory + ainvoke_with_fallback
│   ├── vector_store.py           # Supabase pgvector + Gemini embeddings
│   ├── events.py                 # WebSocket event types incl. REPORT_TOKEN
│   ├── stream_manager.py         # Per-request async queue + emit_sync
│   ├── stream_context.py         # ContextVar for synthesizer stream injection
│   └── logging_config.py         # Structured JSON logging + per-request trace IDs
├── agents/
│   ├── state.py                  # LangGraph ResearchState TypedDict
│   ├── tools.py                  # 7 parallel API fetchers in ThreadPoolExecutor
│   ├── graph.py                  # Supervisor graph + async _wrap_node
│   └── nodes/
│       ├── supervisor.py         # Deterministic routing via completed_steps
│       ├── search_agent.py       # Parallel search + LLM fallback
│       ├── rag_agent.py          # pgvector cosine search with user_id filter
│       ├── summarizer_agent.py   # Structured extraction with citation tagging
│       └── synthesizer_agent.py  # Cross-paper synthesis + live token streaming
├── api/
│   ├── routes.py                 # REST + WebSocket + history endpoints
│   └── auth_routes.py            # /token (fallback), /me, /logout
└── papermind-react/
    └── src/
        ├── lib/supabase.js       # Supabase singleton client
        ├── hooks/useResearch.js  # Auth + research + history hooks
        └── components/
            ├── LoginModal.jsx    # Sign in / Sign up / Google OAuth
            ├── Sidebar.jsx       # Provider status, KB stats, PDF upload
            ├── SearchBar.jsx     # Search + history dropdown + follow-up toggle
            ├── AgentTerminal.jsx # Live pipeline event log with progress bar
            ├── ReportView.jsx    # Streaming markdown renderer + follow-up questions
            └── ThemeToggle.jsx
```
 
---
 
## API Reference
 
| Endpoint | Method | Auth | Description |
|---|---|---|---|
| `/api/v1/auth/token` | POST | — | Fallback dev login |
| `/api/v1/auth/me` | GET | ✓ | Current user info |
| `/api/v1/ws/research?token=` | WS | ✓ | Stream agent events + report tokens |
| `/api/v1/upload-pdf` | POST | ✓ | Ingest PDF into pgvector (per-user) |
| `/api/v1/vector-store/stats` | GET | ✓ | Per-user chunk count |
| `/api/v1/vector-store/clear` | DELETE | ✓ | Clear user's KB + delete files |
| `/api/v1/history` | GET | ✓ | Get user's research history |
| `/api/v1/history/{id}` | GET | ✓ | Fetch specific session report |
| `/api/v1/history/{id}` | DELETE | ✓ | Delete session from history |
| `/api/v1/providers` | GET | — | Active LLM provider + configured |
| `/api/v1/health` | GET | — | Health check + version |
| `/docs` | GET | — | Swagger UI |
 
---
 
## Local Setup
 
```bash
# 1. Clone
git clone https://github.com/yourusername/papermind
cd papermind
 
# 2. Install backend
pip install -r requirements.txt
 
# 3. Configure
cp .env.example .env
# Required: GROQ_API_KEY, GEMINI_API_KEY, SUPABASE_URL, SUPABASE_KEY, SUPABASE_JWT_SECRET
 
# 4. Run Supabase SQL migrations (see /sql/schema.sql)
 
# 5. Run backend
python main.py
 
# 6. Run frontend
cd papermind-react
npm install
cp .env.example .env  # VITE_API_URL, VITE_SUPABASE_URL, VITE_SUPABASE_ANON_KEY
npm run dev
```
 
Login with `demo / papermind2024` in dev mode — no Supabase config required.
 
---
 
## Environment Variables
 
### Backend `.env`
```
LLM_PROVIDER=groq
GROQ_API_KEY=your_groq_key
GEMINI_API_KEY=your_gemini_key
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_KEY=your-service-role-key
SUPABASE_JWT_SECRET=your-jwt-secret
SECRET_KEY=any-32-char-string
APP_ENV=development
```
 
### Frontend `papermind-react/.env`
```
VITE_API_URL=http://localhost:8000
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=your-anon-key
```
 
---
 
## Deployment
 
| Service | Platform | Notes |
|---|---|---|
| Backend | Render (free) | Auto-deploy from GitHub, WebSocket supported |
| Frontend | Vercel (free) | Auto-deploy from GitHub, CDN edge |
| Auth + DB | Supabase (free) | Google OAuth, pgvector, 500MB storage |
| LLM | Groq free tier | 14,400 req/day, llama-3.3-70b |
| Fallback LLM | Gemini free tier | Auto-failover when Groq hits limits |
| Embeddings | Gemini API (free) | 1,500 req/day, gemini-embedding-001 |
 
---
 
## License
 
MIT — see [LICENSE](LICENSE)
 
---
 
*Built to demonstrate production-grade multi-agent AI systems, async backend engineering, and real-world deployment constraints at zero cost.*
