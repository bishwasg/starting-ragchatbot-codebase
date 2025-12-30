# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A full-stack RAG (Retrieval-Augmented Generation) chatbot for answering questions about course materials. Combines ChromaDB vector storage with Anthropic's Claude API for semantic search and intelligent responses.

## Development Commands

**Important: This project uses `uv` for package management. Always use `uv` commands instead of `pip`.**

### Running the Application
```bash
# Quick start
./run.sh

# Manual start
cd backend
uv run uvicorn app:app --reload --port 8000
```

### Setup
```bash
# Install dependencies
uv sync

# Add new package (use uv, NOT pip)
uv add package-name

# Create .env file with required API key
echo "ANTHROPIC_API_KEY=your_key_here" > .env
```

### Access Points
- Web UI: http://localhost:8000
- API docs: http://localhost:8000/docs
- API endpoints:
  - `POST /api/query` - Submit questions
  - `GET /api/courses` - Get course statistics

## Architecture

### RAG Pipeline Flow

```
User Query (frontend)
    ↓
FastAPI endpoint (/api/query)
    ↓
RAGSystem.query() - orchestrates the flow
    ↓
AIGenerator.generate_response() - calls Claude API with tool use
    ↓
Claude decides to use CourseSearchTool
    ↓
VectorStore.search() - semantic search in ChromaDB
    ↓
Returns relevant chunks to Claude
    ↓
Claude synthesizes answer from retrieved context
    ↓
Response with source attribution returned to user
```

### Key Components

**RAGSystem** (`rag_system.py`) - Main orchestrator
- Coordinates all components (document processor, vector store, AI generator, session manager)
- Handles document ingestion and query processing
- Manages conversation sessions

**VectorStore** (`vector_store.py`) - ChromaDB wrapper
- Two collections: `course_catalog` (metadata) and `course_content` (chunks)
- Persistent storage at `./chroma_db`
- Uses SentenceTransformer (`all-MiniLM-L6-v2`) for embeddings

**DocumentProcessor** (`document_processor.py`) - Text processing
- Parses course documents with expected format (title, instructor, lessons)
- Sentence-based chunking: 800 chars with 100 char overlap
- Adds contextual prefixes to chunks for better retrieval

**AIGenerator** (`ai_generator.py`) - Claude integration
- Uses tool use pattern for semantic search
- Model: `claude-sonnet-4-20250514`
- Manages conversation history and tool execution

**ToolManager & CourseSearchTool** (`search_tools.py`)
- Defines search tool schema for Claude
- Executes vector searches and formats results
- Tracks sources for attribution

### Data Models

**Course** - Complete course with title, instructor, link, and lessons
**Lesson** - Individual lesson with number, title, and optional link
**CourseChunk** - Text segment for vector storage with course/lesson metadata

### Document Processing Pipeline

1. **Startup**: `@app.on_event("startup")` in `app.py` triggers document loading
2. **Folder Processing**: `RAGSystem.add_course_folder()` scans `../docs` directory
3. **Deduplication**: Checks existing course titles in ChromaDB to skip re-processing
4. **Per Document**:
   - Parse metadata (lines 1-3: title, link, instructor)
   - Find lesson markers (`Lesson N:` pattern)
   - Chunk lesson content (sentence-based with overlap)
   - Add context prefix: `"Course {title} Lesson {n} content: {text}"`
5. **Storage**:
   - Course metadata → `course_catalog` collection
   - Chunks with embeddings → `course_content` collection

### Important Implementation Details

**Vector Storage Persistence**
- ChromaDB data persists to disk at `./chroma_db`
- Embeddings are NOT regenerated on restart for existing courses
- However, all documents are re-parsed on startup to check for duplicates (wasteful but safe)

**Session Management**
- In-memory session storage (not suitable for distributed deployment)
- Conversation history limited to last 2 exchanges (MAX_HISTORY=2)
- Session IDs generated and tracked per conversation

**Tool Use Pattern**
- Claude is given a `search_course_materials` tool
- Claude autonomously decides when to search and what query to use
- Search results are injected back into Claude's context
- Sources tracked via ToolManager for attribution

**Expected Document Format**
```
Course Title: [title]
Course Link: [url]
Course Instructor: [name]

Lesson 0: Introduction
Lesson Link: [url]
[content...]

Lesson 1: Next Topic
[content...]
```

### Configuration (`config.py`)

Key settings:
- `CHUNK_SIZE: 800` - Maximum characters per chunk
- `CHUNK_OVERLAP: 100` - Overlap between chunks
- `MAX_RESULTS: 5` - Search results limit
- `MAX_HISTORY: 2` - Conversation exchanges to remember
- `CHROMA_PATH: "./chroma_db"` - Vector storage location
- `EMBEDDING_MODEL: "all-MiniLM-L6-v2"` - SentenceTransformer model

### Common Gotchas

**API Key Issues**
- Application requires Anthropic API key with available credits in `.env`
- Different from Claude Code authentication (they're separate systems)

**Document Loading**
- Documents automatically loaded from `../docs` on startup
- Supported formats: .txt, .pdf, .docx
- Files must follow expected format or parsing will fail silently

**ChromaDB Collections**
- Two separate collections must be maintained together
- Course title is used as unique ID in both collections
- Deleting one without the other causes inconsistency

**Frontend Static Files**
- Custom `DevStaticFiles` class adds no-cache headers for development
- Frontend mounted at root `/` - must be last route registered
