# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the App

```bash
# Start the server (from repo root)
./run.sh

# Or manually
cd backend && uv run uvicorn app:app --reload --port 8000
```

App is served at `http://localhost:8000`. API docs at `http://localhost:8000/docs`.

Requires a `.env` file in the repo root with `ANTHROPIC_API_KEY=...` (see `.env.example`).

## Architecture

This is a RAG (Retrieval-Augmented Generation) chatbot for querying course materials. The backend is a FastAPI app; the frontend is static HTML/JS/CSS served directly by FastAPI.

**Request flow:**
1. User query → `POST /api/query` ([backend/app.py](backend/app.py))
2. `RAGSystem.query()` ([backend/rag_system.py](backend/rag_system.py)) builds a prompt and calls Claude via `AIGenerator`
3. Claude decides whether to invoke the `search_course_content` tool (one call max per query)
4. `CourseSearchTool.execute()` ([backend/search_tools.py](backend/search_tools.py)) queries ChromaDB using local sentence-transformer embeddings
5. Results are fed back to Claude for synthesis; final answer + sources returned to the UI

**Key design decisions:**
- Tool-use is single-turn: Claude gets one search, then synthesizes. There is no multi-step agentic loop — `ai_generator.py` makes at most two Claude API calls per query (initial + after tool result).
- Conversation history is stored in-memory in `SessionManager` ([backend/session_manager.py](backend/session_manager.py)), formatted as a plain string and injected into the system prompt. It is not passed as structured `messages`; sessions are lost on server restart.
- ChromaDB has two collections: `course_catalog` (one doc per course, used for fuzzy course-name resolution) and `course_content` (chunked lesson text, used for semantic search).
- Course title is used as the ChromaDB document ID — duplicate titles will collide.

**Document ingestion:**
- On startup, `app.py` loads all `.txt/.pdf/.docx` files from `../docs/` (relative to `backend/`).
- `DocumentProcessor` ([backend/document_processor.py](backend/document_processor.py)) parses a structured format: first 3 lines are `Course Title:`, `Course Link:`, `Course Instructor:`, followed by `Lesson N: <title>` / `Lesson Link: <url>` / content blocks.
- Chunks are sentence-aware with configurable size (800 chars) and overlap (100 chars) from `config.py`.

**Config** ([backend/config.py](backend/config.py)):
- Model: `claude-sonnet-4-20250514`
- Embeddings: `all-MiniLM-L6-v2` (local, via sentence-transformers)
- ChromaDB persisted to `backend/chroma_db/`

## Adding a New Tool

1. Create a class in [backend/search_tools.py](backend/search_tools.py) that extends `Tool` and implements `get_tool_definition()` (Anthropic tool schema) and `execute(**kwargs)`.
2. Register it in `RAGSystem.__init__()` via `self.tool_manager.register_tool(your_tool)`.
3. If the tool produces sources to display in the UI, add a `last_sources` list attribute — `ToolManager.get_last_sources()` and `reset_sources()` pick it up automatically.
