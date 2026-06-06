# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

```bash
# From the project root (Git Bash on Windows)
./run.sh

# Or manually
cd backend
uv run uvicorn app:app --reload --port 8000
```

The server starts at `http://localhost:8000`. On startup it auto-indexes all `.txt/.pdf/.docx` files in `../docs/` into ChromaDB, skipping courses already present.

## Environment Setup

```bash
uv sync                         # install dependencies
cp .env.example .env            # then add ANTHROPIC_API_KEY
```

Always use `uv` to run Python commands — never use `pip` or `python` directly:

```bash
uv run python script.py         # run a script
uv run uvicorn app:app ...      # run the server
uv add <package>                # add a dependency
```

No test suite or linter is configured in this project.

## Architecture

This is a full-stack RAG chatbot. The FastAPI backend (`backend/`) serves the vanilla JS frontend (`frontend/`) as static files and exposes two API endpoints: `POST /api/query` and `GET /api/courses`.

### Query pipeline (the critical path)

1. **`app.py`** receives `POST /api/query`, creates a session if none exists, delegates to `RAGSystem.query()`
2. **`rag_system.py`** fetches conversation history, then calls `AIGenerator.generate_response()` with the `search_course_content` tool attached
3. **`ai_generator.py`** makes the first Claude API call. If Claude invokes the tool (`stop_reason == "tool_use"`), `_handle_tool_execution()` runs it and makes a second Claude call to synthesize the final answer
4. **`search_tools.py`** → **`vector_store.py`**: `CourseSearchTool` calls `VectorStore.search()`, which (a) resolves a fuzzy course name via the `course_catalog` ChromaDB collection, then (b) retrieves top-5 chunks from the `course_content` collection using `all-MiniLM-L6-v2` embeddings
5. Sources are collected from `tool.last_sources`, saved to session history, and returned to the frontend

### ChromaDB collections

Two persistent collections in `./chroma_db` (relative to `backend/`):
- `course_catalog` — one document per course (title, instructor, lesson list as JSON), used for fuzzy course-name resolution
- `course_content` — chunked lesson text with metadata (`course_title`, `lesson_number`, `chunk_index`), used for semantic retrieval

### Course document format

Files in `docs/` must follow this header convention for `DocumentProcessor` to parse them correctly:

```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 1: <lesson title>
Lesson Link: <url>
<lesson content...>

Lesson 2: ...
```

`course_title` is used as the ChromaDB document ID — it must be unique across all loaded files.

### Adding a new search tool

Implement the `Tool` ABC in `search_tools.py` (provide `get_tool_definition()` returning an Anthropic tool schema and `execute(**kwargs)` returning a string), then register it with `tool_manager.register_tool(your_tool)` in `RAGSystem.__init__()`.

### Key configuration (`backend/config.py`)

| Setting | Default | Effect |
|---|---|---|
| `ANTHROPIC_MODEL` | `claude-sonnet-4-20250514` | Claude model used for generation |
| `EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | Sentence-transformer for ChromaDB embeddings |
| `CHUNK_SIZE` | 800 chars | Max chunk size for document splitting |
| `CHUNK_OVERLAP` | 100 chars | Overlap between consecutive chunks |
| `MAX_RESULTS` | 5 | Top-k chunks returned per search |
| `MAX_HISTORY` | 2 | Conversation turns retained per session |
| `CHROMA_PATH` | `./chroma_db` | ChromaDB persistence path (relative to `backend/`) |
