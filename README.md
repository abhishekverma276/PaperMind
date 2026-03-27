# 🧠 PaperMind — Multi-Agent AI Research Assistant

> Production-grade multi-agent system that autonomously searches 7 academic
> databases, synthesizes findings across 40+ papers, and delivers structured
> research reports with citation grounding — in minutes.

[![Python](https://img.shields.io/badge/Python-3.11-3776ab?logo=python&logoColor=white)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![LangGraph](https://img.shields.io/badge/LangGraph-1.1-6f42c1)](https://langchain-ai.github.io/langgraph)
[![React](https://img.shields.io/badge/React-18-61dafb?logo=react&logoColor=white)](https://react.dev)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

---

## 🎯 What it does

PaperMind deploys a team of AI agents that:

1. **Search** — queries Arxiv, OpenAlex, EuropePMC, Crossref, PubMed,
   Semantic Scholar, and CORE in parallel
2. **Retrieve** — semantically retrieves relevant chunks from uploaded PDFs
3. **Summarize** — extracts structured data from each source with citation numbers
4. **Synthesize** — produces a research report with cross-paper analysis,
   methodology distribution, contradiction detection, and confidence scoring

Every claim in the report is grounded with inline citations `[1][3]`.

---

## 🏗️ Architecture
```
User Query ──→ FastAPI + WebSocket
                    │
                    ▼
         LangGraph Supervisor (deterministic routing)
                    │
        ┌───────────┼────────────┬──────────────┐
        ▼           ▼            ▼              ▼
   Search Agent  RAG Agent  Summarizer    Synthesizer
   (7 APIs in    (ChromaDB   (structured   (cross-paper
    parallel)    + cosine    extraction,   synthesis,
                 threshold)  citations)    token stream)
        │           │            │              │
        └───────────┴────────────┴──────────────┘
                    │
              Supabase (sessions)
              ChromaDB (vectors)
              Ollama/Groq/Gemini (LLM)
```

---

## ⚡ Tech Stack

| Layer | Technology | Why |
|---|---|---|
| Agent orchestration | LangGraph 1.1 | Stateful graph, supervisor pattern |
| Backend | FastAPI + WebSocket | Async, streaming, production-grade |
| Auth | JWT + Supabase | Session persistence, revocation |
| LLM | Ollama → Groq → Gemini | Auto-fallback chain, zero cost dev |
| Vector store | ChromaDB + sentence-transformers | Local, no API cost |
| PDF parsing | PyMuPDF | Fast, handles academic PDFs |
| Academic APIs | Arxiv, OpenAlex, EuropePMC, Crossref, PubMed, S2, CORE | All free |
| Frontend | React + Vite | Token streaming, theme toggle, auth |
| Logging | python-json-logger | Structured JSON, request trace IDs |
| Deployment | Render + Netlify | Free tier, auto-deploy |

---

## 🚀 Quick Start
```bash
# 1. Clone
git clone https://github.com/yourusername/papermind
cd papermind

# 2. Install
pip install -r requirements.txt

# 3. Configure
cp .env.example .env
# Add GROQ_API_KEY (free at console.groq.com)
# Add SUPABASE_URL + SUPABASE_KEY (free at supabase.com) — optional

# 4. Start local LLM (optional — for free local inference)
ollama pull llama3.2:3b
ollama serve

# 5. Start API
python main.py

# 6. Start UI (separate terminal)
cd papermind-react
npm install && npm run dev
```

Open [localhost:3000](http://localhost:3000)
Login: `demo` / `papermind2024`

---

## 🔑 Key Engineering Decisions

**Why LangGraph over CrewAI?**
LangGraph exposes the graph explicitly — you write the nodes, edges, and
state schema. This means full control over routing logic, state persistence,
and debugging. CrewAI abstracts this away, which is fine for demos but
limits production customisation. The supervisor uses deterministic
rule-based routing (not LLM-based) to eliminate hallucination in agent
orchestration.

**Why `asyncio.to_thread` for LLM calls?**
LangChain's sync `llm.invoke()` blocks the event loop. Wrapping it in
`asyncio.to_thread` offloads the blocking call to a thread pool, keeping
FastAPI's event loop free to handle other WebSocket messages and HTTP
requests concurrently.

**Why cosine distance threshold in RAG?**
Without a relevance threshold, ChromaDB always returns the top-n chunks
regardless of semantic similarity. A threshold of 0.55 means off-topic
queries (e.g. biology questions when a crypto PDF is indexed) return zero
chunks — preventing cross-domain contamination in the report.

**Why ContextVar for stream injection?**
The StreamManager needs to reach the synthesizer node, but LangGraph state
is typed and shouldn't carry non-serialisable objects. ContextVar provides
per-request context that flows through async boundaries and thread pools
without polluting the state schema.

---

## 📡 API Reference

| Endpoint | Method | Auth | Description |
|---|---|---|---|
| `/api/v1/auth/token` | POST | — | Get JWT token |
| `/api/v1/auth/logout` | POST | ✓ | Revoke session |
| `/api/v1/auth/me` | GET | ✓ | Current user |
| `/api/v1/research` | POST | ✓ | Run pipeline, return report |
| `/api/v1/ws/research?token=` | WS | ✓ | Stream agent events + tokens |
| `/api/v1/upload-pdf` | POST | ✓ | Ingest PDF into ChromaDB |
| `/api/v1/vector-store/stats` | GET | — | KB chunk count |
| `/api/v1/vector-store/clear` | DELETE | ✓ | Clear KB + delete files |
| `/api/v1/providers` | GET | — | Active LLM provider |
| `/api/v1/health` | GET | — | Health check |
| `/docs` | GET | — | Swagger UI |

---

## 🚢 Deployment

**Backend → Render (free tier)**
```bash
# render.yaml already configured
# 1. Push to GitHub
# 2. Connect repo on render.com
# 3. Set env vars: GROQ_API_KEY, SUPABASE_URL, SUPABASE_KEY, SECRET_KEY
```

**Frontend → Netlify (free tier)**
```bash
cd papermind-react
npm run build
# Drag dist/ folder to netlify.com/drop
# Or connect GitHub for auto-deploy
# Set env var: VITE_API_URL=https://your-render-url.onrender.com
```

**Switch to production LLM (one line):**
```bash
LLM_PROVIDER=groq  # 14,400 req/day free, <5s response time
```

---

## 📁 Project Structure
```
papermind/
├── main.py                    # FastAPI + middleware
├── core/
│   ├── config.py              # Settings from .env
│   ├── auth.py                # JWT + session validation
│   ├── database.py            # Supabase client + fallback
│   ├── llm.py                 # Multi-provider LLM factory
│   ├── vector_store.py        # ChromaDB + embeddings
│   ├── events.py              # WebSocket event types
│   ├── stream_manager.py      # Per-request event queue
│   ├── stream_context.py      # ContextVar for stream injection
│   └── logging_config.py      # Structured JSON logging
├── agents/
│   ├── state.py               # LangGraph shared state
│   ├── tools.py               # 7 academic API fetchers
│   ├── graph.py               # Agent graph + node wrappers
│   └── nodes/
│       ├── supervisor.py      # Deterministic router
│       ├── search_agent.py    # Parallel search
│       ├── rag_agent.py       # ChromaDB retrieval
│       ├── summarizer_agent.py # Structured extraction
│       └── synthesizer_agent.py # Cross-paper synthesis
├── api/
│   ├── routes.py              # REST + WebSocket endpoints
│   └── auth_routes.py         # Login / logout / me
└── papermind-react/           # React frontend
    └── src/
        ├── App.jsx
        ├── hooks/useResearch.js
        └── components/
            ├── Sidebar.jsx
            ├── SearchBar.jsx
            ├── AgentTerminal.jsx
            ├── ReportView.jsx
            ├── ThemeToggle.jsx
            └── LoginModal.jsx
```

---

## 🧪 Running Tests
```bash
pytest tests/ -v
```

---

## 📝 License

MIT — see [LICENSE](LICENSE)

---

*Built as a portfolio project demonstrating multi-agent AI systems,
async backend engineering, and production-grade Python.*
