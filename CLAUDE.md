# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

RAG chatbot for querying course materials using ChromaDB vector storage and Anthropic's Claude API.

## Setup & Commands

**Install dependencies:**
```bash
uv sync
```

**Run application:**
```bash
./run.sh
# Or: cd backend && uv run uvicorn app:app --reload --port 8000
```

**Environment:** Create `.env` file with `ANTHROPIC_API_KEY=your_key_here`

**Access:** http://localhost:8000 (web UI), http://localhost:8000/docs (API docs)

## Architecture

### Two-Phase Query Processing

Each user query triggers **two sequential Claude API calls** (implemented in `ai_generator.py:43-135`):

**Phase 1:** Claude receives query + tool definitions → decides whether to search → returns `stop_reason="tool_use"`

**Phase 2:** Tool executes (searches ChromaDB) → Claude receives search results → synthesizes final answer

This pattern is coordinated by `RAGSystem.query()` which orchestrates: session history retrieval → AI generation with tools → source tracking → history update.

### Component Coordination

**RAGSystem** (`rag_system.py`) instantiates and coordinates:
- `DocumentProcessor` - parses course files into chunks
- `VectorStore` - manages dual ChromaDB collections
- `AIGenerator` - handles Claude API with tool execution
- `ToolManager` - registers tools and tracks sources
- `SessionManager` - maintains in-memory conversation history

### ChromaDB Dual-Collection Pattern

**Two collections serve different purposes:**

1. **`course_catalog`** - Course metadata (1 doc per course)
   - ID: course title (used for deduplication)
   - Metadata: lessons (JSON), instructor, link

2. **`course_content`** - Searchable chunks (N docs per course)
   - Metadata: course_title, lesson_number, chunk_index
   - Embeddings: 384-dim vectors from `all-MiniLM-L6-v2`

**Search flow:** Query → embedding → semantic search → course name resolution → filter content → top K results

### Document Processing Details

**Expected format in `/docs` files:**
```
Course Title: [title]
Course Link: [url]
Course Instructor: [name]

Lesson N: [title]
Lesson Link: [optional]
[content]
```

**Chunking:** Sentence-based (not fixed-length) using regex to detect sentence boundaries while handling abbreviations. Chunks get context prefix: `"Course {title} Lesson {number} content: {chunk}"`

**Deduplication:** Course title = unique ID. Re-processing same title skips if exists (`rag_system.py:87`).

**Startup:** `app.py:88-99` loads all documents from `../docs` on server start.

### Session Management

**In-memory only** - lost on restart. Keeps last `MAX_HISTORY * 2` messages (default: 4 messages = 2 exchanges). History injected into system prompt for context.

### Key Configuration Parameters

**`backend/config.py`:**
- `CHUNK_SIZE: 800` - Max chunk characters
- `CHUNK_OVERLAP: 100` - Overlap between chunks
- `MAX_RESULTS: 5` - Top K from vector search
- `MAX_HISTORY: 2` - Conversation exchanges remembered
- `CHROMA_PATH: "./chroma_db"` - Vector DB location

**Important:** Changing `EMBEDDING_MODEL` requires rebuilding entire ChromaDB (incompatible embeddings).

### Tool-Based Search Pattern

**Adding new tools:**
1. Inherit from `Tool` (`search_tools.py:6-17`)
2. Implement `get_tool_definition()` - Anthropic tool schema
3. Implement `execute(**kwargs)` - tool logic
4. Register: `self.tool_manager.register_tool(new_tool)`

**Current tool:** `CourseSearchTool` supports parameters: `query` (required), `course_name` (optional), `lesson_number` (optional)

### Critical Implementation Details

**AI Generator** (`ai_generator.py`):
- System prompt is static class variable to avoid rebuilding
- Temperature: 0 (consistency), max_tokens: 800 (concise)
- Constraint: "One search per query maximum" to control costs

**Vector Store** (`vector_store.py`):
- Course name resolution uses semantic search on catalog
- Unified `search()` interface handles filtering
- Returns `SearchResults` object with documents, metadata, distances

**Frontend Contract** (`app.py:56-86`):
- POST `/api/query`: `{query, session_id}` → `{answer, sources, session_id}`
- GET `/api/courses`: → `{total_courses, course_titles}`

### Important Constraints

1. Sessions are volatile (in-memory only)
2. Course titles must be unique (used as IDs)
3. CORS is wide open (`allow_origins=["*"]`) - no auth
4. Two API calls per query affects latency/cost
5. ChromaDB persists to disk but sessions don't
