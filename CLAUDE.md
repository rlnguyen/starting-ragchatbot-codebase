# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Course Materials RAG System — a full-stack app that answers questions about course transcripts using ChromaDB for semantic retrieval and Claude for generation, with a FastAPI backend and a static JS frontend.

## Commands

- Use `uv` for all package management — never `pip` directly (e.g. `uv add <pkg>`, `uv remove <pkg>`, `uv run <cmd>`).
- Install dependencies: `uv sync`
- Run the app: `./run.sh`, or manually: `cd backend && uv run uvicorn app:app --reload --port 8000`
- Requires `ANTHROPIC_API_KEY` in a `.env` file at the repo root (see `.env.example`)
- Web UI: `http://localhost:8000` — FastAPI's auto-generated API docs: `http://localhost:8000/docs`
- No test suite or lint/format tooling (pytest, ruff, black, etc.) is configured in this repo.

## Architecture

This is **tool-calling RAG**, not always-retrieve RAG: whether a question triggers a vector search is decided by Claude itself, mid-request, based on an instruction in `ai_generator.py`'s `SYSTEM_PROMPT` ("general knowledge → answer directly" vs. "course-specific → search first"). There is no `if` branch anywhere in the orchestration code that makes this call — it lives entirely in the model's reasoning.

**Request flow**: `frontend/script.js` → `POST /api/query` (`backend/app.py`) → `RAGSystem.query()` (`backend/rag_system.py`) is the orchestrator — it builds the prompt, pulls session history, and calls `AIGenerator.generate_response()` with the search tool attached.

**The two-call tool loop** (`backend/ai_generator.py`): the first Claude call always includes `tools` + `tool_choice: auto`. If `stop_reason == "tool_use"`, `_handle_tool_execution()` runs the requested tool via `ToolManager` and makes a **second** call with the tool results appended and no `tools` key — that second response is the final answer. If Claude answers directly, there's no second call.

**Tools** (`backend/search_tools.py`): `ToolManager`/`CourseSearchTool` registers the only tool, `search_course_content`. `CourseSearchTool` also tracks `last_sources` from the most recent search; `RAGSystem` reads and resets this after each query — it's how source citations reach the frontend even though the tool itself just returns a formatted string back into the Claude conversation.

**Storage — two ChromaDB collections** (`backend/vector_store.py`), one client:
- `course_catalog` — one document per course (title only), used purely to resolve a fuzzy name like "MCP" to the exact stored course title via a 1-result semantic query (`_resolve_course_name`).
- `course_content` — the actual chunks, filtered by exact `course_title`/`lesson_number` metadata and ranked by embedding similarity.

A course-filtered search always does both steps: resolve, then filter — never a single query.

**Ingestion** (`backend/document_processor.py`): on FastAPI startup, `RAGSystem.add_course_folder()` loads every file in `../docs`, skipping courses already present in `course_catalog`. Expected file format: first lines are `Course Title:` / `Course Link:` / `Course Instructor:`, then body text split on `Lesson N: <title>` markers. Chunking (`chunk_text()`) is sentence-aware, packing to `CHUNK_SIZE` chars with `CHUNK_OVERLAP` overlap. Only the first chunk of each lesson is prefixed with lesson context (`"Lesson N content: ..."`) — except the last lesson in a file, where *every* chunk gets that prefix. Most chunks rely on ChromaDB metadata, not embedded text, to carry their lesson context.

**Config** (`backend/config.py`): all tunables live in one dataclass — `ANTHROPIC_MODEL`, `EMBEDDING_MODEL` (`all-MiniLM-L6-v2`), `CHUNK_SIZE` (800), `CHUNK_OVERLAP` (100), `MAX_RESULTS` (5), `MAX_HISTORY` (2 exchanges), `CHROMA_PATH`.

**Session state** (`backend/session_manager.py`): plain in-memory dict keyed by session id — history does not persist across a server restart.
