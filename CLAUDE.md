# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a RAG (Retrieval-Augmented Generation) chatbot system for querying educational course materials. It combines semantic search with Claude AI to provide intelligent, context-aware answers about course content.

**Tech Stack:**
- Backend: FastAPI + Python 3.13
- Vector DB: ChromaDB with Sentence Transformers (all-MiniLM-L6-v2)
- AI: Anthropic Claude API with tool use
- Frontend: Vanilla JavaScript + HTML/CSS

## Development Commands

**IMPORTANT**: Always use `uv` for all Python operations. Do NOT use `pip` directly.

### Environment Setup
```bash
# Install dependencies
uv sync

# Create .env file (required)
cp .env.example .env
# Then add your ANTHROPIC_API_KEY to .env
```

### Running the Application
```bash
# Quick start (from root)
./run.sh

# Manual start (from root)
cd backend
uv run uvicorn app:app --reload --port 8000

# Run any Python script
uv run python script.py
uv run python -m module_name

# NEVER use: python script.py, python -m uvicorn, or pip install
# ALWAYS use: uv run python <script> or uv run <command>
```

Access at: http://localhost:8000

### Package Management
```bash
# Add new dependency
uv add package-name

# Remove dependency
uv remove package-name

# Update dependencies
uv sync

# DO NOT use pip install, pip freeze, or requirements.txt
# This project uses uv for all dependency management
```

## Architecture Overview

### RAG Pipeline Flow

The system follows a multi-stage RAG architecture:

1. **Query Entry** → Frontend sends user query + session_id to `/api/query`
2. **RAG Orchestrator** (`rag_system.py`) coordinates all components
3. **AI Decision** → Claude decides whether to search (via tool use)
4. **Tool Execution** (if needed):
   - **Course Resolution**: Semantic search in `course_catalog` collection to match course names
   - **Content Search**: Filtered semantic search in `course_content` collection
   - **Result Formatting**: Returns chunks with course/lesson context + tracked sources
5. **AI Generation** → Claude synthesizes final answer using search results
6. **Response** → Returns answer + sources to frontend

### Key Architectural Patterns

#### Two-Phase Claude API Interaction
When tools are available, the system makes **two API calls**:
1. First call: Claude decides if search is needed and generates tool use request
2. Second call: Claude receives tool results and generates final answer

Check `ai_generator.py:_handle_tool_execution()` for implementation.

#### Dual Vector Collections
ChromaDB uses **two separate collections**:
- `course_catalog`: Course metadata for semantic course name matching
- `course_content`: Actual course chunks for content retrieval

This enables fuzzy course name matching (e.g., "MCP" → "MCP: Build Rich-Context AI Apps with Anthropic").

#### Context-Augmented Chunking
Chunks are prefixed with context during processing:
- `"Course {title} Lesson {N} content: {chunk}"`

This improves retrieval accuracy by embedding course/lesson context directly in the searchable text.

See `document_processor.py:process_course_document()` lines 186, 234.

#### Session-Based Conversation History
Sessions maintain conversation context with configurable history limits (`MAX_HISTORY=2` in config).

History is formatted as text and prepended to system prompt for each request, not passed as messages array. See `ai_generator.py` lines 61-64.

#### Source Tracking via Tool State
Sources are tracked in `CourseSearchTool.last_sources` during search execution, then retrieved by `ToolManager.get_last_sources()` after AI generation completes.

This decouples source tracking from the AI generation flow.

## Component Responsibilities

### Backend Core (`backend/`)

**`rag_system.py`** - Main orchestrator
- Initializes all components (document processor, vector store, AI generator, tools)
- Exposes `query()` method that coordinates the full RAG pipeline
- Manages document loading and course analytics

**`ai_generator.py`** - Claude API wrapper
- Handles both single-turn and tool-use conversations
- Manages tool execution loop (request → execute → follow-up)
- System prompt is static class variable for efficiency

**`vector_store.py`** - ChromaDB interface
- Main entry point: `search(query, course_name, lesson_number)`
- Implements course name resolution via semantic search
- Builds ChromaDB filters from search parameters
- Returns `SearchResults` dataclass (not raw ChromaDB format)

**`search_tools.py`** - Tool definitions for Claude
- `CourseSearchTool`: Implements search logic and result formatting
- `ToolManager`: Registers tools and executes them by name
- Tools follow `Tool` ABC protocol (get_tool_definition, execute)

**`document_processor.py`** - Document parsing
- Expects specific format: `Course Title:`, `Course Link:`, `Course Instructor:`, then `Lesson N:` markers
- Chunks text by sentences with configurable overlap
- Returns `(Course, List[CourseChunk])` tuple

**`session_manager.py`** - Conversation state
- In-memory session storage (not persisted)
- Maintains last N exchanges per session
- Returns formatted string history, not message objects

**`config.py`** - Configuration
- Loads from `.env` file via python-dotenv
- Key settings: `CHUNK_SIZE=800`, `CHUNK_OVERLAP=100`, `MAX_RESULTS=5`, `MAX_HISTORY=2`

**`app.py`** - FastAPI application
- Two main endpoints: `/api/query` (POST), `/api/courses` (GET)
- Loads documents from `../docs/` on startup
- Serves frontend as static files from `../frontend/`

### Frontend (`frontend/`)

**`script.js`** - Main application logic
- Session management (creates session on first message)
- Message rendering with markdown support (marked.js)
- Collapsible source attribution display

**`index.html`** - Web interface
- Chat interface with sidebar showing course statistics
- Suggested question buttons for quick queries

## Document Format

Course documents in `docs/` must follow this structure:

```
Course Title: [title]
Course Link: [url]
Course Instructor: [name]

Lesson 0: [title]
Lesson Link: [url]
[content...]

Lesson 1: [title]
Lesson Link: [url]
[content...]
```

The parser uses regex to extract metadata and lesson markers. See `document_processor.py` lines 116-243.

## Important Configuration

**Embedding Model**: `all-MiniLM-L6-v2` (Sentence Transformers)
- Fast, lightweight model suitable for semantic search
- Change via `EMBEDDING_MODEL` in config.py

**Claude Model**: `claude-sonnet-4-20250514`
- Configured in `config.py:ANTHROPIC_MODEL`
- Requires `ANTHROPIC_API_KEY` environment variable

**ChromaDB Location**: `backend/chroma_db/`
- Persistent storage, created automatically
- Delete this directory to force rebuild from documents

## Data Flow Summary

```
User Query
  ↓
FastAPI (/api/query)
  ↓
RAGSystem.query()
  ├→ SessionManager (get history)
  ├→ AIGenerator.generate_response()
  │   ├→ Claude API (1st call - decision)
  │   ├→ ToolManager.execute_tool()
  │   │   └→ CourseSearchTool.execute()
  │   │       └→ VectorStore.search()
  │   │           ├→ Resolve course name (semantic)
  │   │           └→ Search content (filtered)
  │   └→ Claude API (2nd call - with results)
  ├→ ToolManager.get_last_sources()
  └→ SessionManager.add_exchange()
  ↓
QueryResponse (answer + sources)
  ↓
Frontend (markdown rendering)
```

## Windows Development

Windows users should use **Git Bash** to run shell scripts (`./run.sh`). Alternatively, run commands manually from PowerShell/CMD:

```bash
cd backend
uv run uvicorn app:app --reload --port 8000
```
