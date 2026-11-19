# Document Intelligence Platform: RAG & Agentic AI Chatbot

A full-stack **retrieval-augmented generation (RAG)** platform for document intelligence. Users upload PDF, DOCX and TXT files, then search them semantically, ask questions and get answers that cite the exact source passage, and run structured AI workflows such as contract clause extraction or resume-to-job-description matching.

The full technical specification (database schema, API design, RAG pipeline and deployment plan) is in [ARCHITECTURE.md](ARCHITECTURE.md).

## Project status

The backend foundation is built and runs with one `docker compose` command. The RAG pipeline, chat and workflow endpoints are in progress.

| Area | Status |
|---|---|
| Docker Compose stack: PostgreSQL + pgvector, Redis, MinIO, API, Celery worker | Done |
| Configuration with typed settings (`pydantic-settings`) | Done |
| Database schema and Alembic migration (7 tables, `vector` column for embeddings) | Done |
| JWT authentication helpers (access + refresh tokens, bcrypt password hashing) | Done |
| S3-compatible file storage service (upload, download, presigned URLs, delete) | Done |
| Celery worker wiring | Done |
| Swappable LLM and embedding provider interfaces | Done |
| Health endpoint and unit tests for health and security | Done |
| Ingestion pipeline: parse → chunk → embed → store | In progress |
| Semantic search and citation-grounded chat (streamed answers) | In progress |
| REST endpoints for auth, collections, documents, search, chat and workflows | In progress |
| Six AI workflows (listed below) | Planned |
| Next.js frontend | Planned |

## Planned features

- **Semantic search** across uploaded documents with `pgvector` similarity search
- **Citation-grounded chat:** answers stream back with references to the source document and page
- **Asynchronous ingestion:** uploads return right away while Celery parses, chunks and embeds the file in the background
- **Collections** to organize documents into folders
- **Six structured AI workflows**, each returning validated JSON:
  1. Document summarization
  2. Multi-document Q&A
  3. Contract clause extraction
  4. Resume ↔ job description match
  5. Meeting notes digest
  6. Comparative analysis across documents
- **Swappable providers:** the LLM (Groq by default, OpenAI supported) and the embedding model are chosen in configuration, not hard-coded

## Tech stack

| Layer | Technology |
|---|---|
| API | Python 3.11, FastAPI, Pydantic |
| Database | PostgreSQL 16 + pgvector, SQLAlchemy 2.0 (async), Alembic |
| LLM orchestration | LangChain |
| LLM | Groq (Llama 3.3 70B) by default; OpenAI optional |
| Embeddings | `sentence-transformers` (`all-MiniLM-L6-v2`, 384 dimensions), run locally |
| Background jobs | Celery + Redis |
| File storage | MinIO locally; any S3-compatible store in production |
| Document parsing | `pypdf`, `python-docx`, `unstructured` |
| Auth | JWT (`python-jose`), bcrypt (`passlib`) |
| Testing | pytest, httpx |
| Containers | Docker, Docker Compose |

## Architecture

```
            ┌──────────────┐
  client ──▶│  FastAPI API │──────────────┐
            └──────┬───────┘              │
                   │ enqueue job          │ read / write
                   ▼                      ▼
            ┌──────────────┐      ┌──────────────────────┐
            │    Redis     │      │ PostgreSQL + pgvector│
            └──────┬───────┘      │ documents, chunks,   │
                   │              │ embeddings, chats    │
                   ▼              └──────────▲───────────┘
            ┌──────────────┐                 │
            │ Celery worker│─────────────────┘
            │ parse, chunk,│
            │ embed        │──────▶ MinIO / S3 (original files)
            └──────────────┘
```

## Getting started

### Prerequisites

- Docker and Docker Compose
- Python 3.11+ (only to run tests outside Docker)

### Run locally

```bash
git clone https://github.com/Gargi4344/Document-Intelligence-Platform----RAG-Agentic-AI-Chatbot.git
cd Document-Intelligence-Platform----RAG-Agentic-AI-Chatbot

# Create the backend environment file
cp backend/.env.example backend/.env
# Add a free Groq API key (https://console.groq.com) to GROQ_API_KEY for LLM features

docker compose up --build
```

On first start, the API container applies database migrations automatically.

| Service | URL |
|---|---|
| API | http://localhost:8001 |
| Interactive API docs (Swagger) | http://localhost:8001/docs |
| Health check | http://localhost:8001/api/health |
| MinIO console (user and password: `minioadmin`) | http://localhost:9011 |
| PostgreSQL | `localhost:5434` (user and password: `postgres`, database `docintel`) |

### Run tests

```bash
cd backend
pip install -r requirements.txt
pytest
```

## Project structure

```
.
├── ARCHITECTURE.md          # Technical spec: schema, API design, RAG pipeline
├── docker-compose.yml       # Postgres + pgvector, Redis, MinIO, API, worker
└── backend/
    ├── Dockerfile
    ├── alembic.ini
    ├── requirements.txt
    ├── app/
    │   ├── main.py          # FastAPI app, CORS, health endpoint
    │   ├── api/             # Routers: auth, collections, documents, search, chat, workflows
    │   ├── core/            # Settings, JWT security, dependencies
    │   ├── db/              # Async session and Alembic migrations
    │   ├── models/          # SQLAlchemy models
    │   ├── schemas/         # Pydantic request/response schemas
    │   ├── services/        # Storage, ingestion, embeddings, RAG, LLM providers, workflows
    │   └── workers/         # Celery app and tasks
    └── tests/               # pytest suite
```
