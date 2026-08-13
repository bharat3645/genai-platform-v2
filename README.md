# GenAI Platform

A microservices-based GenAI platform that combines five modules — multi-PDF chat with GraphRAG, ATS resume analysis, an autonomous research assistant, natural-language-to-SQL, and a standalone knowledge-graph engine — behind a single FastAPI API gateway, with a React/TypeScript frontend.

This is **not** a Supabase app. The frontend talks to a self-hosted API gateway (`http://localhost:8000` by default) that proxies to five independent FastAPI microservices backed by PostgreSQL, Qdrant, Neo4j, Redis, and Ollama.

## Architecture

```
                    React Frontend (Vite + TypeScript)
                              │  HTTPS + JWT
                              ▼
                    API Gateway (FastAPI, :8000)
              JWT auth · rate limiting · SSE proxy · correlation IDs
                              │
        ┌──────────┬──────────┼──────────┬──────────┐
        ▼          ▼          ▼          ▼          ▼
     PDF-RAG      ATS      Research     SQL      KG Engine
    :8001      :8002       :8003      :8004    (in PDF-RAG)
   (Qdrant +   (6-agent   (7-stage    (triple-
    Neo4j)     pipeline)   HTN)        layer
                                       safety)
        │          │          │          │
        ▼          │          ▼          ▼
     Qdrant        │      Tavily /   PostgreSQL
     Neo4j         │       Serper
        │          │          │          │
        └──────────┴──────────┴──────────┘
                    │
        PostgreSQL · Redis · Ollama (LLM + embeddings)
```

`services/` holds five independent FastAPI apps plus a `shared/` package (config, Pydantic models, a LiteLLM-backed `LLMProvider` for talking to Ollama, and the Postgres init schema):

| Service | Port | Responsibility |
|---|---|---|
| `services/api-gateway` | 8000 | Single entry point: JWT issuing/verification, request proxying to the four downstream services, SSE stream proxying, CORS, rate limiting (`slowapi`), correlation-ID request logging, aggregated `/health` |
| `services/pdf-rag-service` | 8001 | PDF ingestion (PyMuPDF text extraction → `tiktoken` chunking → embeddings → Qdrant) and hybrid retrieval (vector search + Neo4j graph traversal); also hosts the standalone Knowledge Graph engine (`kg_engine.py`) used by Module 5 |
| `services/ats-agent-service` | 8002 | 6-agent resume analyzer (Coordinator → Keyword/Format/Content/Job-Match in parallel → Improvement → Synthesis), with a programmatic 40/30/30 weighted score |
| `services/research-service` | 8003 | 7-stage HTN research pipeline (Plan → Search → Filter → Summarize → Verify → Synthesize → Cite) using Tavily/Serper web search with an LLM fallback when no search API key is set |
| `services/sql-service` | 8004 | Natural-language-to-SQL over the live Postgres schema, with triple-layer safety: `sqlglot` AST check (SELECT-only), regex DML/DDL blocklist, and enforced `LIMIT 1000`, executed in a read-only transaction |

The frontend (`src/`) is a Vite + React 18 + TypeScript SPA. All backend calls go through `src/lib/api.ts`, a single HTTP client that hits `${VITE_API_URL}/api/*` (default `http://localhost:8000`) with a JWT bearer token stored in `localStorage`, plus a small SSE helper for streaming endpoints (PDF query, ATS analysis, research progress).

Also present: `k8s/` (namespace, config, stateful data-layer, and service manifests for a production deployment) and `monitoring/` (Prometheus + Grafana provisioning, with its own `docker-compose.monitoring.yml`).

For more detail, see [`docs/`](docs/) — `architecture.md`, one write-up per module (`module-1-pdf-graphrag.md` … `module-5-knowledge-graph.md`), `api-reference.md`, `deployment.md`, and `configuration.md`.

## What's real vs. what's mocked

Being precise, since this matters:

- **PDF-RAG (Module 1 + 2), ATS analyzer (Module 3), Research assistant (Module 4), Text-to-SQL (Module 5), Knowledge Graph engine — all real.** These services make genuine LLM calls (via LiteLLM to a local Ollama model, swappable to OpenAI/Anthropic through env vars), perform real PDF parsing, real vector search in Qdrant, real graph writes/traversal in Neo4j, real Postgres schema introspection, and real SQL execution with the safety layers described above. Nothing in `services/pdf-rag-service`, `services/ats-agent-service`, `services/research-service`, or `services/sql-service` returns hardcoded/canned data.
- **API Gateway auth is currently mocked.** `POST /api/auth/register` and `POST /api/auth/login` (`services/api-gateway/main.py`) are explicitly marked `# MOCK AUTH` in the code: they accept any email/password, never touch the database, and always return a token for a hardcoded `mock-user-id`. The pieces for real auth already exist — a `users` table with a `hashed_password` column (`services/shared/init_db.sql`), and `passlib[bcrypt]` is a declared dependency — but the handlers don't use them yet. Wiring real registration/login (hash + persist + verify against `users`) is the main gap between this and a real multi-user deployment.
- **Web search in the Research service** falls back to asking the LLM to *generate* plausible-looking search results if neither `TAVILY_API_KEY` nor `SERPER_API_KEY` is set (`search_web()` in `services/research-service/main.py`). With a key configured it does real web search via Tavily or Serper.

## Quick start

### Prerequisites

- Docker & Docker Compose v2
- Node.js 18+ (for frontend development)
- ~8 GB RAM free (Ollama + Postgres + Qdrant + Neo4j + Redis)

### 1. Clone and configure

```bash
git clone https://github.com/bharat3645/genai-platform-v2.git
cd genai-platform-v2
cp .env.example .env
# Edit .env: set JWT_SECRET, and optionally TAVILY_API_KEY / SERPER_API_KEY
```

### 2. Start the backend

```bash
docker-compose up -d
```

This brings up Postgres, Redis, Qdrant, Neo4j, Ollama, and all five FastAPI services (see `docker-compose.yml`).

### 3. Pull the LLM models

```bash
docker exec -it genai-ollama ollama pull llama3
docker exec -it genai-ollama ollama pull nomic-embed-text
```

### 4. Verify the backend is up

```bash
curl http://localhost:8000/health
# { "status": "ok", "services": { "gateway": "ok", "rag": "ok", "ats": "ok", "research": "ok", "sql": "ok" }, "timestamp": "..." }
```

### 5. Start the frontend

```bash
npm install
npm run dev
# → http://localhost:5173
```

By default the frontend talks to `http://localhost:8000` (see `VITE_API_URL` in `.env.example` / `src/lib/api.ts`).

### Other scripts

```bash
npm run build       # production build → dist/
npm run lint         # eslint
npm run typecheck    # tsc --noEmit
```

## Production deployment

Kubernetes manifests live in `k8s/` (`namespace.yaml`, `config.yaml`, `databases.yaml`, `services.yaml`) and a Prometheus/Grafana stack lives in `monitoring/`. See [`docs/deployment.md`](docs/deployment.md) for the full walkthrough, including scaling notes and Grafana default credentials.

## Project structure

```
genai-platform-v2/
├── services/
│   ├── api-gateway/          # Auth, proxying, rate limiting, SSE
│   ├── pdf-rag-service/      # PDF ingestion, hybrid retrieval, KG engine
│   ├── ats-agent-service/    # 6-agent resume analyzer
│   ├── research-service/     # 7-stage HTN research pipeline
│   ├── sql-service/          # Text-to-SQL with triple-layer safety
│   └── shared/                # Config, Pydantic models, LLM provider, DB init SQL
├── src/                       # React + TypeScript frontend
├── docs/                      # Architecture, per-module, API reference, deployment, config
├── k8s/                       # Kubernetes manifests
├── monitoring/                # Prometheus + Grafana
├── docker-compose.yml         # Local dev orchestration
└── .env.example                # Configuration template
```

## License

MIT — see [LICENSE](LICENSE).
