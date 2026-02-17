# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A full-stack Retrieval-Augmented Generation (RAG) chatbot for querying course materials. Uses ChromaDB for vector storage, Anthropic Claude (with tool calling) for AI responses, and a vanilla HTML/JS frontend.

## Commands

**Always use `uv` to manage all dependencies** (never pip). Use `uv add <package>` to add dependencies and `uv run` to execute Python commands.

```bash
# Install dependencies
uv sync

# Add a new dependency
uv add <package>

# Run the app (from repo root)
./run.sh

# Or manually (must cd into backend/)
cd backend && uv run uvicorn app:app --reload --port 8000
```

- Web UI: http://localhost:8000
- API docs: http://localhost:8000/docs
- Requires `ANTHROPIC_API_KEY` in a `.env` file at the repo root (see `.env.example`)

There are no test or lint commands configured for this project.

## Architecture

```
Frontend (vanilla HTML/JS/CSS) → FastAPI Backend → RAG System Orchestrator
                                                      ├─ DocumentProcessor  (parse & chunk docs)
                                                      ├─ VectorStore        (ChromaDB)
                                                      ├─ AIGenerator        (Claude API + tool use)
                                                      ├─ ToolManager        (search_course_content tool)
                                                      └─ SessionManager     (conversation history)
```

**Key design decision:** The system uses Claude's **tool calling** to perform semantic search rather than doing a naive vector lookup. Claude decides when and how to call `search_course_content`, which queries ChromaDB with optional course name and lesson filters.

### Backend (`backend/`)

- **app.py** — FastAPI entry point. Serves the frontend as static files and exposes `POST /api/query` and `GET /api/courses`. On startup, loads all documents from `docs/` via `RAGSystem.add_course_folder()`.
- **rag_system.py** — Orchestrator that wires together all components. Entry point for queries is `RAGSystem.query()`.
- **document_processor.py** — Parses plain-text course files (expected headers: `Course Title:`, `Course Instructor:`, `Lesson N: Title`) and splits content into sentence-aware chunks.
- **vector_store.py** — ChromaDB wrapper with two collections: `course_catalog` (metadata) and `course_content` (chunked text). Uses `all-MiniLM-L6-v2` embeddings.
- **ai_generator.py** — Anthropic API client. Handles the tool-use loop: sends messages to Claude, processes tool calls, feeds results back until a final text response.
- **search_tools.py** — Defines the `search_course_content` tool schema and execution logic. Tracks sources from the last search for attribution.
- **session_manager.py** — Stores per-session conversation history (default max 2 exchanges).
- **models.py** — Pydantic models: `Course`, `Lesson`, `CourseChunk`.
- **config.py** — Dataclass-based config. Key tunables: `CHUNK_SIZE` (800), `CHUNK_OVERLAP` (100), `MAX_RESULTS` (5), `MAX_HISTORY` (2), `ANTHROPIC_MODEL` (claude-sonnet-4-20250514).

### Frontend (`frontend/`)

Vanilla HTML/JS/CSS. Uses `marked.js` for Markdown rendering. Features a chat interface, course sidebar, and suggested question buttons. All served as static files by FastAPI.

### Data (`docs/`)

Plain-text course scripts. Processed at server startup and stored in ChromaDB (persisted to `chroma_db/` on disk). Existing courses are skipped on restart (deduplication by title).

## Tech Stack

- **Python 3.13+**, managed with **uv**
- FastAPI + Uvicorn
- ChromaDB 1.0.15 with sentence-transformers embeddings
- Anthropic SDK (Claude claude-sonnet-4-20250514 with tool use)
