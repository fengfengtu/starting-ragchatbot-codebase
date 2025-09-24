# RAG System Query Processing Flow

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                USER QUERY                                        │
└─────────────────────────┬───────────────────────────────────────────────────────┘
                          │
                          v
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           1. API ENDPOINT                                        │
│                         app.py:query_documents()                                 │
│  ┌─────────────────────────────────────────────────────────────────────────────┐│
│  │ • Receives POST /api/query                                                  ││
│  │ • Creates session_id if not provided                                        ││
│  │ • Calls rag_system.query()                                                 ││
│  └─────────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────┬───────────────────────────────────────────────────────┘
                          │
                          v
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      2. RAG SYSTEM ORCHESTRATOR                                  │
│                       rag_system.py:query()                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐│
│  │ • Formats query as AI prompt                                               ││
│  │ • Retrieves conversation history                                           ││
│  │ • Prepares tools for AI (search_course_content)                           ││
│  │ • Calls ai_generator.generate_response()                                   ││
│  └─────────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────┬───────────────────────────────────────────────────────┘
                          │
                          v
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        3. AI GENERATOR                                           │
│                   ai_generator.py:generate_response()                            │
│  ┌─────────────────────────────────────────────────────────────────────────────┐│
│  │ • Sends query to Claude AI with system prompt                              ││
│  │ • System prompt: "Use search tool for course questions"                    ││
│  │ • Claude decides: General knowledge OR Course-specific                     ││
│  └─────────────────────────────────────────────────────────────────────────────┘│
└─────────────┬───────────────────────────────────────┬───────────────────────────┘
              │                                       │
              v                                       v
┌─────────────────────────┐                 ┌─────────────────────────┐
│    GENERAL KNOWLEDGE    │                 │    COURSE-SPECIFIC      │
│      (Direct Answer)    │                 │   (Tool Execution)      │
└─────────────────────────┘                 └─────────┬───────────────┘
              │                                       │
              │                                       v
              │                             ┌─────────────────────────────────────┐
              │                             │        4. TOOL EXECUTION            │
              │                             │   ai_generator.py:_handle_tool_     │
              │                             │            _execution()             │
              │                             │ ┌─────────────────────────────────┐ │
              │                             │ │ • Parse tool calls from Claude  │ │
              │                             │ │ • Execute search_course_content │ │
              │                             │ │ • Collect tool results          │ │
              │                             │ └─────────────────────────────────┘ │
              │                             └─────────┬───────────────────────────┘
              │                                       │
              │                                       v
              │                             ┌─────────────────────────────────────┐
              │                             │         5. COURSE SEARCH            │
              │                             │    search_tools.py:execute()        │
              │                             │ ┌─────────────────────────────────┐ │
              │                             │ │ • Semantic search on query      │ │
              │                             │ │ • Optional course/lesson filter │ │
              │                             │ │ • Format results with context   │ │
              │                             │ └─────────────────────────────────┘ │
              │                             └─────────┬───────────────────────────┘
              │                                       │
              │                                       v
              │                             ┌─────────────────────────────────────┐
              │                             │        6. VECTOR STORE              │
              │                             │      vector_store.py:search()       │
              │                             │ ┌─────────────────────────────────┐ │
              │                             │ │ • ChromaDB similarity search    │ │
              │                             │ │ • Sentence transformer embeddings│ │
              │                             │ │ • Return ranked results + metadata│ │
              │                             │ └─────────────────────────────────┘ │
              │                             └─────────┬───────────────────────────┘
              │                                       │
              │                                       │
              │                ┌──────────────────────┘
              │                │
              │                v
              │      ┌─────────────────────────────────────┐
              │      │    RESULTS BACK TO CLAUDE AI       │
              │      │ • Tool results sent to Claude      │
              │      │ • Claude generates final response   │
              │      └─────────┬───────────────────────────┘
              │                │
              └────────────────┴─────────────────────────────┐
                                                             │
                                                             v
                                    ┌─────────────────────────────────────┐
                                    │        7. RESPONSE ASSEMBLY         │
                                    │         Back to RAG System          │
                                    │ ┌─────────────────────────────────┐ │
                                    │ │ • Extract sources from tools    │ │
                                    │ │ • Update conversation history   │ │
                                    │ │ • Return (answer, sources)      │ │
                                    │ └─────────────────────────────────┘ │
                                    └─────────┬───────────────────────────┘
                                              │
                                              v
                                    ┌─────────────────────────────────────┐
                                    │        8. API RESPONSE              │
                                    │       Back to app.py                │
                                    │ ┌─────────────────────────────────┐ │
                                    │ │ • Create QueryResponse object   │ │
                                    │ │ • Include answer, sources,      │ │
                                    │ │   session_id                    │ │
                                    │ └─────────────────────────────────┘ │
                                    └─────────┬───────────────────────────┘
                                              │
                                              v
                                    ┌─────────────────────────────────────┐
                                    │           USER SEES RESPONSE         │
                                    │       • AI-generated answer         │
                                    │       • Source references          │
                                    │       • Session maintained         │
                                    └─────────────────────────────────────┘
```

## Key Components:

**Session Management**: Maintains conversation context across queries
**Tool Decision**: Claude AI intelligently decides when to search vs. answer directly
**Vector Search**: Semantic similarity using ChromaDB and sentence transformers
**Context Preservation**: Each search result includes course and lesson metadata
**Source Tracking**: UI displays which documents were used to generate the answer