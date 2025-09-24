# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Running the Application
```bash
# Quick start (recommended)
./run.sh

# Manual start
cd backend && uv run uvicorn app:app --reload --port 8000

# Install dependencies
uv sync
```

### Environment Setup
- Requires Python 3.13+ and `uv` package manager
- Must set `ANTHROPIC_API_KEY` in `.env` file
- Application runs on http://localhost:8000

## Architecture Overview

This is a **RAG (Retrieval-Augmented Generation) System** for querying course materials using semantic search and AI responses.

### Core Components

**Backend Architecture (`backend/`):**
- **FastAPI Application** (`app.py`) - API server with CORS, serves frontend
- **RAG System** (`rag_system.py`) - Main orchestrator coordinating all components
- **AI Generator** (`ai_generator.py`) - Claude AI integration with tool calling
- **Vector Store** (`vector_store.py`) - ChromaDB persistence with dual collections
- **Document Processor** (`document_processor.py`) - Text chunking and course parsing
- **Search Tools** (`search_tools.py`) - Tool-based semantic search interface
- **Session Manager** (`session_manager.py`) - Conversation history management

**Frontend (`frontend/`):**
- Simple HTML/CSS/JavaScript interface
- Served as static files through FastAPI

### Data Flow Architecture

1. **Document Loading**: On startup, processes all files in `docs/` folder
2. **Vector Storage**: Two ChromaDB collections:
   - `course_catalog` - Course metadata for name resolution
   - `course_content` - Text chunks with embeddings for semantic search
3. **Query Processing**:
   - API receives user query → RAG System → AI Generator
   - Claude decides whether to search course content or answer directly
   - Tool-based search executes semantic similarity search
   - AI synthesizes final response with sources

### Document Format Expected

Course documents should follow this structure:
```
Course Title: [title]
Course Link: [url]
Course Instructor: [instructor]

Lesson 0: Introduction
Lesson Link: [optional lesson url]
[lesson content...]

Lesson 1: Next Topic
[lesson content...]
```

### Key Configuration

**Models Used:**
- AI: `claude-sonnet-4-20250514`
- Embeddings: `all-MiniLM-L6-v2` (SentenceTransformers)

**Processing Settings:**
- Chunk size: 800 characters
- Chunk overlap: 100 characters
- Max search results: 5
- Conversation history: 2 messages

### Vector Store Details

**Persistent Storage**: ChromaDB at `./chroma_db/`
**Smart Loading**: Avoids reprocessing existing courses on restart
**Dual Collections**: Separate course metadata from content for efficient filtering
**Context Enhancement**: Chunks include course and lesson context for better retrieval

### API Endpoints

- `POST /api/query` - Process user questions with RAG
- `GET /api/courses` - Get course statistics
- `GET /` - Serve web interface

### Session Management

Each query can include a `session_id` for conversation continuity. If not provided, a new session is created automatically.