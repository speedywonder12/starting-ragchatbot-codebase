# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Install dependencies**
```bash
uv sync
```

**Run the server**
```bash
./run.sh
# or manually:
cd backend && uv run uvicorn app:app --reload --port 8000
```

**Add a dependency**
```bash
uv add <package>
```

Always use `uv` to run the server, manage dependencies, and execute Python files — never use `pip` or `python` directly.

```bash
uv run python <file>.py   # run any Python file
uv add <package>          # add a dependency
uv run uvicorn ...        # run the server
```

There are no tests in this codebase.

## Architecture

This is a full-stack RAG (Retrieval-Augmented Generation) chatbot. The frontend is plain HTML/JS served as static files by the FastAPI backend. All Python code lives in `backend/`, course documents in `docs/`, and the ChromaDB database is written to `backend/chroma_db/` at runtime.

### Request flow

1. Browser POSTs `{ query, session_id }` to `POST /api/query`
2. `app.py` hands it to `RAGSystem.query()`
3. `RAGSystem` passes the query + conversation history to `AIGenerator.generate_response()`
4. Claude is called with the `search_course_content` tool available (`tool_choice: auto`)
5. If Claude decides to search, `CourseSearchTool.execute()` calls `VectorStore.search()`, which embeds the query and retrieves top-5 chunks from ChromaDB
6. Tool results are appended to the message thread and Claude is called a second time (without tools) to produce the final answer
7. Sources and answer are returned to the browser; the frontend renders the answer as Markdown

### Document ingestion

On startup (`app.py` startup event), all files in `docs/` are ingested via `DocumentProcessor.process_course_document()`:
- Parses course metadata (title, link, instructor) from the first 3 lines
- Splits content by `Lesson N:` markers
- Chunks each lesson's text into ~800-character sentence-based chunks with 100-character overlap
- Stores course metadata in ChromaDB collection `course_catalog` and content chunks in `course_content`
- Already-indexed courses (matched by title) are skipped on subsequent startups

### Document format

Course files must follow this structure:
```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 1: <title>
Lesson Link: <url>
<content>

Lesson 2: <title>
...
```

### Key configuration (`backend/config.py`)

| Setting | Value | Purpose |
|---|---|---|
| `ANTHROPIC_MODEL` | `claude-sonnet-4-20250514` | Model used for all generation |
| `EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | SentenceTransformers model for embeddings |
| `CHUNK_SIZE` | 800 | Max characters per chunk |
| `CHUNK_OVERLAP` | 100 | Character overlap between chunks |
| `MAX_RESULTS` | 5 | Top-k chunks returned per search |
| `MAX_HISTORY` | 2 | Conversation exchanges retained per session |

### Component responsibilities

- **`rag_system.py`** — orchestrator; wires all components together
- **`document_processor.py`** — parses course files and chunks text
- **`vector_store.py`** — ChromaDB wrapper; handles embedding, storage, and semantic search across two collections (`course_catalog`, `course_content`)
- **`ai_generator.py`** — Anthropic SDK client; manages the two-call tool-use loop
- **`search_tools.py`** — defines the `search_course_content` tool and `ToolManager` registry
- **`session_manager.py`** — in-memory conversation history keyed by session ID
- **`models.py`** — Pydantic models: `Course`, `Lesson`, `CourseChunk`
