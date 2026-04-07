# 🧠 PaperMind — Multi-Agent AI Research Assistant
 
> A production-grade multi-agent system that autonomously searches 7 academic databases in parallel, synthesizes findings across 40+ papers, and delivers structured research reports with citation grounding, contradiction detection, and confidence scoring — streamed token by token in real time.
 
**Live Demo:** [paper-mind-deployment.vercel.app](https://paper-mind-deployment.vercel.app)
**Backend API:** [paperminddeployment.onrender.com/docs](https://paperminddeployment.onrender.com/docs)
 
[![Python](https://img.shields.io/badge/Python-3.11-3776ab?logo=python&logoColor=white)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![LangGraph](https://img.shields.io/badge/LangGraph-1.1-6f42c1)](https://langchain-ai.github.io/langgraph)
[![React](https://img.shields.io/badge/React-18-61dafb?logo=react&logoColor=white)](https://react.dev)
[![Supabase](https://img.shields.io/badge/Supabase-Auth%20%2B%20pgvector-3ecf8e?logo=supabase&logoColor=white)](https://supabase.com)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)
 
---
 
## What it does
 
PaperMind deploys a team of 4 specialist AI agents orchestrated by a LangGraph supervisor:
 
1. **Search Agent** — queries Arxiv, OpenAlex, EuropePMC, Crossref, PubMed, Semantic Scholar, and CORE in parallel using `ThreadPoolExecutor`; deduplicates across sources; gracefully skips rate-limited APIs
2. **RAG Agent** — semantically retrieves relevant chunks from user-uploaded PDFs using Supabase pgvector + Gemini embeddings; cosine similarity threshold filtering prevents cross-domain contamination
3. **Summarizer Agent** — extracts structured citation-ready data from each source with exact metrics, methodology, and relevance scoring
4. **Synthesizer Agent** — produces a cross-paper report with methodology distribution, numbered inline citations, contradiction detection, confidence scoring, and real-time token streaming
 
Every factual claim in the report is grounded with inline citations `[1][3]`. The report appears word-by-word as it generates.
 
---
 
## Architecture
 
```
User Query → FastAPI + WebSocket (Render)
                    │
                    ▼
       LangGraph Supervisor (deterministic routing)
                    │
     ┌──────────────┼──────────────┬─────────────────┐
     ▼              ▼              ▼                  ▼
Search Agent    RAG Agent    Summarizer          Synthesizer
(7 APIs,       (pgvector +   (structured         (cross-paper
 parallel,      Gemini        extraction,         synthesis,
 dedup)         embeddings,   citations,          token stream,
                threshold     exact metrics)      debates,
                filter)                           confidence)
     │              │              │                  │
     └──────────────┴──────────────┴──────────────────┘
                    │
           Supabase (Auth + pgvector)
           Groq / Gemini (LLM)
           React + Vite (Vercel)
```
 
---
 
## Tech Stack
 
| Layer | Technology | Decision rationale |
|---|---|---|
| Agent orchestration | LangGraph 1.1 | Explicit state graph, deterministic supervisor routing, no magic |
| Backend | FastAPI + WebSocket | Async, streaming-first, production-grade |
| Auth | Supabase Auth | Google OAuth + email/password, ES256 JWT, session management |
| Vector store | Supabase pgvector | Eliminates sentence-transformers (400MB) — fits Render 512MB free tier |
| Embeddings | Gemini text-embedding-004 | Free API, 768-dim, no local model needed |
| LLM | Groq llama-3.3-70b | Free tier, <5s response, auto-fallback to Gemini |
| PDF parsing | PyMuPDF | Fast, handles academic PDFs |
| Academic APIs | Arxiv, OpenAlex, EuropePMC, Crossref, PubMed, S2, CORE | All free |
| Frontend | React + Vite | Token streaming, dark/light theme, Google OAuth |
| Logging | python-json-logger | Structured JSON, per-request trace IDs via ContextVar |
| Deployment | Render + Vercel | Free tier, auto-deploy from GitHub |
 
**Total monthly cost: $0**
 
---
 
## Key Engineering Decisions
 
**Why LangGraph over CrewAI?**
LangGraph gives you the full graph — nodes, edges, state schema, routing logic. The supervisor uses deterministic rule-based routing via a `completed_steps` list instead of LLM-based routing, eliminating hallucination in agent orchestration. CrewAI abstracts this away which limits production control.
 
**Why `asyncio.to_thread` for LLM calls?**
LangChain's sync `llm.invoke()` blocks the event loop. Every agent node uses `asyncio.to_thread` to offload to a thread pool. The synthesizer uses `llm.stream()` inside the thread and emits tokens immediately via `emit_sync`, preventing Render's 90-second WebSocket idle timeout during long synthesis.
 
**Why ContextVar for stream injection?**
The `StreamManager` needs to reach the synthesizer node, but LangGraph state is typed and shouldn't carry non-serialisable objects. `ContextVar` provides per-request context that flows through `async` boundaries and thread pools without polluting the state schema.
 
**Why cosine distance threshold in RAG?**
Without a relevance threshold, pgvector always returns top-n chunks regardless of semantic similarity. A threshold of 0.45 means off-topic queries return zero chunks, preventing cross-domain contamination across multiple uploaded PDFs.
 
**Why Supabase pgvector over ChromaDB?**
ChromaDB + sentence-transformers requires ~400MB memory — exceeds Render's 512MB free tier. Supabase pgvector + Gemini embeddings API uses ~0MB locally, reducing total memory from ~700MB to ~250MB.
 
**Why ES256 JWT verification via JWKS?**
Modern Supabase projects issue asymmetric ES256 tokens. The backend fetches the public key from Supabase's JWKS endpoint, cached via `@lru_cache`. Zero shared secrets, supports Google OAuth tokens natively.
 
---
 
## Project Structure
 
```
papermind/
├── main.py                     # FastAPI app + RequestIdMiddleware
├── core/
│   ├── config.py               # Pydantic settings from .env
│   ├── auth.py                 # ES256/HS256 JWT verification via JWKS
│   ├── database.py             # Supabase client + in-memory fallback
│   ├── llm.py                  # Multi-provider factory + ainvoke_with_fallback
│   ├── vector_store.py         # Supabase pgvector + Gemini embeddings
│   ├── events.py               # WebSocket event types incl. REPORT_TOKEN
│   ├── stream_manager.py       # Per-request async queue + emit_sync
│   ├── stream_context.py       # ContextVar for synthesizer stream injection
│   └── logging_config.py       # Structured JSON logging + trace IDs
├── agents/
│   ├── state.py                # LangGraph ResearchState TypedDict
│   ├── tools.py                # 7 parallel API fetchers in ThreadPoolExecutor
│   ├── graph.py                # Supervisor graph + async _wrap_node
│   └── nodes/
│       ├── supervisor.py       # Deterministic rule-based routing
│       ├── search_agent.py     # Parallel search + LLM tool call fallback
│       ├── rag_agent.py        # pgvector cosine similarity retrieval
│       ├── summarizer_agent.py # Structured extraction with exact metrics
│       └── synthesizer_agent.py # Cross-paper synthesis + live token streaming
├── api/
│   ├── routes.py               # REST + WebSocket endpoints
│   └── auth_routes.py          # /token (fallback), /me, /logout
└── papermind-react/
    └── src/
        ├── lib/supabase.js     # Supabase singleton client
        ├── hooks/useResearch.js # Auth + research hooks
        └── components/
            ├── LoginModal.jsx   # Sign in / Sign up / Google OAuth
            ├── Sidebar.jsx      # Provider status, KB stats, PDF upload
            ├── AgentTerminal.jsx # Live pipeline event log with progress bar
            ├── ReportView.jsx   # Streaming markdown renderer + cursor
            ├── SearchBar.jsx
            └── ThemeToggle.jsx
```
 
---
 
## API Reference
 
| Endpoint | Method | Auth | Description |
|---|---|---|---|
| `/api/v1/auth/token` | POST | — | Fallback dev login |
| `/api/v1/auth/me` | GET | ✓ | Current user info |
| `/api/v1/ws/research?token=` | WS | ✓ | Stream agent events + report tokens |
| `/api/v1/upload-pdf` | POST | ✓ | Ingest PDF into pgvector |
| `/api/v1/vector-store/stats` | GET | — | KB chunk count |
| `/api/v1/vector-store/clear` | DELETE | ✓ | Clear KB + delete files |
| `/api/v1/providers` | GET | — | Active LLM provider |
| `/api/v1/health` | GET | — | Health check |
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
 
# 4. Run backend
python main.py
 
# 5. Run frontend
cd papermind-react
npm install && npm run dev
```
 
Login with `demo / papermind2024` in dev mode (no Supabase config needed).
 
---
 
## Environment Variables
 
### Backend `.env`
```
LLM_PROVIDER=groq
GROQ_API_KEY=your_key
GEMINI_API_KEY=your_key
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
 
## License
 
MIT — see [LICENSE](LICENSE)
 
---
 
*Built to demonstrate production-grade multi-agent AI systems, async backend engineering, and real-world deployment constraints.*
