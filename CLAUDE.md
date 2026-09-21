# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Quick Start Commands

### Environment Setup
```bash
# Install dependencies
uv sync

# Set up environment variables
cp .env.example .env
# Then edit .env and add your ANTHROPIC_API_KEY
```

### Running the Application
```bash
# Start the server (from project root)
./run.sh

# Or manually start backend server (from backend directory)
uv run uvicorn app:app --reload --port 8000
```

The application will be available at:
- Web Interface: `http://localhost:8000`
- API Docs: `http://localhost:8000/docs`

### Development
```bash
# Run Python directly for testing modules
uv run python backend/main_rag.py

# Add a new course document folder
uv run python -c "from backend.rag_system import RAGSystem; from backend.config import config; rag = RAGSystem(config); rag.add_course_folder('docs')"
```

## Architecture Overview

This is a **Retrieval-Augmented Generation (RAG) system** that answers questions about course materials by:

1. **Ingestion**: Processing PDF/DOCX/TXT documents and splitting them into semantic chunks
2. **Indexing**: Storing chunks in ChromaDB with embeddings for fast semantic search
3. **Retrieval**: Using Claude's tool-use to retrieve relevant context for queries
4. **Generation**: Generating contextual answers using Claude with retrieved documents

### System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         Frontend (vanilla JS)                 │
│                  HTML/CSS/JS web interface                   │
└──────────────────────┬──────────────────────────────────────┘
                       │
        FastAPI HTTP Endpoints (/api/query, /api/courses)
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                     Backend (FastAPI)                        │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ RAGSystem (Main Orchestrator)                        │  │
│  │ - Coordinates document processing, retrieval, and AI │  │
│  └──────────────────────────────────────────────────────┘  │
│   ▲  ▲  ▲                                                   │
│   │  │  └─────────────┬──────────────────┬─────────────┐   │
│   │  │                │                  │             │    │
│  [A] [B]            [C]                [D]           [E]   │
│                                                              │
│ [A] DocumentProcessor → Chunks documents (PDF/DOCX/TXT)    │
│ [B] VectorStore → ChromaDB + embeddings (semantic search)  │
│ [C] AIGenerator → Claude API calls with tool-use          │
│ [D] SearchTools → Tool definitions for semantic queries   │
│ [E] SessionManager → Conversation history tracking         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Key Modules

**`backend/rag_system.py`** — Main orchestrator
- `RAGSystem`: Coordinates all components
- `add_course_document()`: Process single document
- `add_course_folder()`: Bulk document ingestion
- `query()`: Main entry point for processing user queries

**`backend/app.py`** — FastAPI server
- `POST /api/query`: Process user query, return answer + sources
- `GET /api/courses`: Get course statistics
- Serves static frontend from `/frontend`
- Startup event loads initial documents from `/docs`

**`backend/vector_store.py`** — ChromaDB integration
- Manages embeddings and semantic search
- `add_course_metadata()`: Store course info
- `add_course_content()`: Store document chunks
- `search_chunks()`: Semantic similarity search
- `get_existing_course_titles()`: Track loaded courses

**`backend/document_processor.py`** — Document parsing
- `process_course_document()`: Extract text from PDF/DOCX/TXT
- Returns `Course` object with metadata + list of `CourseChunk`s
- Handles chunking: `CHUNK_SIZE=800`, `CHUNK_OVERLAP=100`

**`backend/ai_generator.py`** — Claude integration
- `generate_response()`: Call Claude API with tool definitions
- Handles tool-use loops for semantic search
- Maintains conversation history from `SessionManager`

**`backend/search_tools.py`** — Tool definitions
- `CourseSearchTool`: Tool definition that Claude can invoke
- Returns relevant document chunks via vector search
- Sources tracked for response attribution

**`backend/session_manager.py`** — Conversation tracking
- Creates and maintains session IDs
- Stores conversation history (configured for 2 messages)
- Used for multi-turn context

**`backend/models.py`** — Data structures
- `Course`: Document metadata (id, title, filename)
- `Lesson`: Lesson info within a course
- `CourseChunk`: Text chunk with metadata (course_id, section, position)

## Configuration

**`backend/config.py`** — Runtime configuration

| Setting | Default | Purpose |
|---------|---------|---------|
| `ANTHROPIC_API_KEY` | From `.env` | Claude API authentication |
| `ANTHROPIC_MODEL` | `claude-sonnet-4-20250514` | Model used for generation |
| `EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | Embedding model for semantic search |
| `CHUNK_SIZE` | 800 | Characters per chunk |
| `CHUNK_OVERLAP` | 100 | Overlap between chunks (context preservation) |
| `MAX_RESULTS` | 5 | Max search results returned per query |
| `MAX_HISTORY` | 2 | Conversation turns to remember |
| `CHROMA_PATH` | `./chroma_db` | ChromaDB storage location |

## Data Flow: Query Processing

When a user submits a query:

1. **Frontend** → `POST /api/query` with user question + session_id
2. **Backend** → `RAGSystem.query()`
3. **AIGenerator** → Calls Claude with search tool definition
4. **Claude's Tool-Use** → Invokes `CourseSearchTool` for relevant chunks
5. **SearchTool** → `VectorStore.search_chunks()` (semantic search)
6. **AIGenerator** → Processes tool results, generates final answer
7. **SessionManager** → Stores Q&A exchange in history
8. **Response** → Returns answer + sources to frontend

## Adding Course Materials

Place course documents (PDF, DOCX, or TXT) in the `/docs` folder. They'll be automatically loaded on startup via the `/startup` event in `app.py`:
- Documents are split into semantic chunks
- Embeddings are generated and stored in ChromaDB
- Courses are de-duplicated by title

Programmatically:
```python
from backend.rag_system import RAGSystem
from backend.config import config

rag = RAGSystem(config)
courses, chunks = rag.add_course_folder('./docs', clear_existing=False)
print(f"Loaded {courses} courses with {chunks} chunks")
```

## Dependencies (via uv)

Key packages (see `pyproject.toml`):
- **chromadb** — Vector database for semantic search
- **anthropic** — Claude API client
- **sentence-transformers** — Embedding model
- **fastapi** — Web framework
- **uvicorn** — ASGI server
- **python-dotenv** — Environment variable loading

All managed via `uv sync` using Python 3.13+.

## Common Development Patterns

**Debugging Tool-Use**: When Claude isn't finding relevant content, check:
1. Document chunking in `DocumentProcessor.process_course_document()`
2. Embedding quality (try different EMBEDDING_MODEL)
3. Semantic search results: `VectorStore.search_chunks()` may return poor matches
4. Tool definition in `SearchTool` — ensure it describes search capability clearly

**Adding a New API Endpoint**: 
1. Define Pydantic models in `app.py`
2. Call `RAGSystem` methods
3. Return response via FastAPI

**Modifying Chunk Size**:
- Increase `CHUNK_SIZE` for broader context (slower search, fewer chunks)
- Decrease for more granular retrieval (more API calls)
- `CHUNK_OVERLAP` prevents context loss at boundaries

**Session Management**: 
- Each query gets a session_id (frontend can provide existing one)
- `MAX_HISTORY=2` means only 2 messages stored (adjust for longer conversations)
- History used to provide context for follow-up questions