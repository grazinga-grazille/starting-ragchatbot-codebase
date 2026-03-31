# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies
uv sync

# Add a dependency
uv add <package>

# Remove a dependency
uv remove <package>

# Run the app (from repo root)
./run.sh

# Run a Python file
uv run python <file>.py

# Run manually (must be run from backend/ directory)
cd backend && uv run uvicorn app:app --reload --port 8000
```

App runs at `http://localhost:8000`. Requires `ANTHROPIC_API_KEY` in a `.env` file at the repo root (see `.env.example`).

There is no test suite.

## Architecture

This is a RAG chatbot that answers questions about course materials stored in `docs/`.

**Request flow:**
1. Frontend (`frontend/`) sends POST `/api/query` with `{query, session_id}`
2. `app.py` delegates to `RAGSystem.query()`
3. `RAGSystem` calls `AIGenerator.generate_response()` with the `search_course_content` tool available
4. Claude decides whether to call the tool; if so, `ToolManager.execute_tool()` runs `CourseSearchTool.execute()`
5. `CourseSearchTool` calls `VectorStore.search()` → ChromaDB semantic search
6. Claude synthesizes the search results into a final response

**Key design decisions:**
- All backend modules run with `backend/` as the working directory (imports are relative, no package structure)
- ChromaDB is stored at `backend/chroma_db/` (path in `config.py` is `./chroma_db`)
- Course documents are loaded on startup via `app.py`'s `startup_event`; already-indexed courses are skipped by title
- ChromaDB uses two collections: `course_catalog` (course-level metadata, one doc per course) and `course_content` (chunked lesson text)
- Course title is used as the ChromaDB document ID — titles must be unique
- Conversation history is stored in-memory only (lost on server restart); `MAX_HISTORY=2` exchange pairs

**Course document format** (for files in `docs/`):
```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>
Lesson 0: <lesson title>
Lesson Link: <url>
<lesson content...>
Lesson 1: <lesson title>
...
```

**Config** (`backend/config.py`): all tuneable parameters (`CHUNK_SIZE`, `CHUNK_OVERLAP`, `MAX_RESULTS`, `MAX_HISTORY`, `ANTHROPIC_MODEL`) are in the `Config` dataclass loaded from `.env`.
