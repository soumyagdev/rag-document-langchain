# RAG Document Assistant

![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat&logo=fastapi&logoColor=white)
![LangChain](https://img.shields.io/badge/LangChain-1C3C3C?style=flat&logo=langchain&logoColor=white)
![ChromaDB](https://img.shields.io/badge/ChromaDB-FF6B35?style=flat)
![OpenAI](https://img.shields.io/badge/OpenAI-412991?style=flat&logo=openai&logoColor=white)
![HTML](https://img.shields.io/badge/HTML-E34F26?style=flat&logo=html5&logoColor=white)

A production-style AI system that lets users query documents in natural language. Built with a clean REST API, LangChain LCEL pipeline, and a dual-store retrieval architecture — engineered for correctness, modularity, and extensibility.

> Designed from the ground up with software engineering principles — separation of concerns, modular components, clean API contracts, and persistent storage.

---

## System Design

```
┌─────────────────────────────────────────────────────────┐
│                    REST API (FastAPI)                    │
│         /upload  /ask  /documents  /inspect  /clear      │
└──────────────┬──────────────────────────┬───────────────┘
               │                          │
       ┌───────▼────────┐        ┌────────▼────────┐
       │  Ingestion      │        │  Query Pipeline  │
       │  Pipeline       │        │  (LCEL Chain)    │
       └───────┬────────┘        └────────┬─────────┘
               │                          │
    ┌──────────▼──────────┐    ┌──────────▼──────────┐
    │   Parent Splitter   │    │   ChromaDB Search    │
    │   2000 char chunks  │    │   Cosine Similarity  │
    └──────────┬──────────┘    └──────────┬──────────┘
               │                          │
    ┌──────────▼──────────┐    ┌──────────▼──────────┐
    │   Child Splitter    │    │  ParentDocument      │
    │   400 char chunks   │    │  Retriever           │
    └──────┬──────┬───────┘    └──────────┬──────────┘
           │      │                        │
    ┌──────▼──┐ ┌─▼──────────┐  ┌─────────▼──────────┐
    │LocalFile│ │  ChromaDB  │  │   LCEL Chain        │
    │Store    │ │  +Embeddings│  │ prompt|llm|parser   │
    │(parents)│ │  (children) │  └────────────────────┘
    └─────────┘ └────────────┘
```

---

## Why This Architecture

### Parent-Child Chunking
Single chunking forces a tradeoff — small chunks improve search precision but hurt answer quality since the LLM lacks surrounding context. Large chunks give the LLM more context but reduce retrieval precision.

The solution: search on small chunks (400 chars), retrieve large parent chunks (2000 chars) for LLM context. Best of both worlds.

### Dual-Store Design
Two stores, each doing what it's optimized for:
- **ChromaDB** — vector similarity search on child chunk embeddings
- **LocalFileStore** — key-value storage for parent chunks (no embedding overhead)

Storing parents in ChromaDB would waste compute embedding text that's never searched. Separation of concerns at the storage layer.

### LCEL Pipeline
The query pipeline is built using LangChain Expression Language — declarative, composable, and streaming-ready:
```python
qa_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)
```
Swapping any component (LLM, retriever, prompt) requires changing one line.

---

## API Design

| Method | Endpoint | Description |
|---|---|---|
| GET | `/` | Health check |
| GET | `/about` | Version info with i18n support |
| POST | `/upload` | Ingest and index a PDF |
| POST | `/ask` | Query with optional source filter |
| GET | `/documents` | List all indexed documents |
| DELETE | `/documents/{filename}` | Remove specific document |
| GET | `/inspect` | Inspect vector store state |
| DELETE | `/clear` | Clear all documents |

---

## Tech Stack

| Layer | Technology |
|---|---|
| API Framework | Python, FastAPI, Uvicorn |
| AI Pipeline | LangChain (LCEL) |
| Vector Store | ChromaDB (child chunks + embeddings) |
| Document Store | LocalFileStore (parent chunks) |
| Embeddings | sentence-transformers (all-MiniLM-L6-v2, local, free) |
| LLM | OpenAI GPT-3.5-turbo |
| PDF Processing | PyMuPDF |
| Frontend | HTML, CSS, JavaScript |

---

## Key Engineering Decisions

**Why FastAPI over Flask?**
Async support, automatic OpenAPI docs, Pydantic validation, and type hints make FastAPI better suited for production AI APIs.

**Why local embeddings over OpenAI embeddings?**
`all-MiniLM-L6-v2` runs locally — zero cost, zero latency on embedding calls, and data never leaves the server.

**Why separate parent and child stores?**
ChromaDB is a vector database optimized for similarity search. LocalFileStore is a key-value store optimized for retrieval by ID. Using the right tool for each job avoids unnecessary embedding computation and storage overhead.

**Why LCEL over a manual chain?**
LCEL gives streaming, async, and LangSmith tracing for free. Swapping components (LLM, retriever, output parser) requires one line change — fully modular.

---

## Setup & Run

```bash
# Install dependencies
pip install -r requirements.txt

# Add your OpenAI API key
echo "OPENAI_API_KEY=your_key_here" > .env

# Run the server
uvicorn main:app --reload
```

Visit `http://127.0.0.1:8000/ui` to use the app.
API docs at `http://127.0.0.1:8000/docs`

---

## Project Structure

```
rag-document-langchain/
├── main.py              # FastAPI app + LangChain RAG pipeline
├── requirements.txt     # All dependencies
├── static/
│   └── index.html       # Frontend UI
├── chroma_db/           # ChromaDB vector store (auto-created)
├── parent_store/        # LocalFileStore for parent chunks (auto-created)
├── .env                 # API keys (not committed)
├── .gitignore
└── README.md
```

---

## What's Next

- Text-to-SQL hybrid — route structured queries to a database, unstructured to RAG
- Table extraction — handle PDFs with tables using pdfplumber
- Authentication — per-user document isolation with JWT
- Batch ingestion — process folders of PDFs automatically
- Docker — containerize for cloud deployment