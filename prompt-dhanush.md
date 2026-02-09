# Desktop AGI Assistant - Technical Specification

## Project Overview
Build a production-ready, agentic desktop application that acts as a personal AI assistant for everyday tasks. The application should be privacy-focused with local LLM processing and modular agent architecture.

## Core Technology Stack
- **GUI Framework**: Flet (cross-platform desktop)
- **LLM**: Ollama (local inference)
  - Primary model: `llama3.1:8b` or `mistral:7b-instruct` for general tasks
  - Embedding model: `nomic-embed-text` for context retrieval
- **Vector Database**: ChromaDB (embedded, for session context storage)
- **Agent Protocol**: MCP (Model Context Protocol) where available
- **Language**: Python 3.10+

## Architecture Requirements

### 1. Session Management
- **Tab-based interface**: Multiple independent sessions (like ChatGPT)
- **Context isolation**: No context sharing between sessions unless explicitly referenced
- **Dual storage system**:
  - **Vector store** (ChromaDB): Semantic search across conversation history
  - **JSON logs**: Complete conversation history per session for exact replay
- **Cross-session reference**: User can explicitly reference previous sessions with phrases like "in the previous tab" or "from yesterday's conversation"

### 2. Context Management (Per Session)
- **Sliding window**: Maintain last N messages in active context
- **Semantic retrieval**: Use ChromaDB to fetch relevant past messages based on current query
- **Context compaction**: Periodically summarize older messages to reduce token usage
- **Coreference resolution**: Handle pronouns ("it", "that", "the document") by maintaining entity tracking
- **Token budget**: Keep context under model's limit (~8K for most 8B models)

### 3. Agentic Loop Architecture
Implement a robust think-act-validate cycle:
```
User Query → Planner (think) → Tool Selection (act) → Validator (validate) → Response
                ↑                                                               ↓
                └──────────────── Retry with feedback ────────────────────────┘
```

**Components**:
- **Planner**: Breaks down user request into steps, identifies which agents needed
- **Executor**: Calls appropriate agents/tools with proper inputs
- **Validator**: Checks if agent outputs are reasonable, retries on failure (max 3 attempts)
- **Synthesizer**: Combines multiple agent outputs into coherent response

## Agent Specifications

### 1. Email Agent (Gmail)
- **MCP Integration**: Use Gmail MCP server if available
- **Capabilities**:
  - Read emails (with filters: unread, from sender, date range, subject keywords)
  - Send emails (with draft preview before sending)
  - Search emails
  - Basic organization (mark as read, archive, label)
- **Authentication**: OAuth 2.0 flow (store tokens securely)
- **Privacy**: Never send email content to external APIs

### 2. Calendar Agent (Google Calendar)
- **MCP Integration**: Use Google Calendar MCP server if available
- **Capabilities**:
  - View events (today, this week, date range)
  - Create events (with conflict detection)
  - Update/delete events
  - Find free time slots
  - Set reminders
- **Authentication**: OAuth 2.0 flow

### 3. File Handling Agent
- **Capabilities**:
  - **Read**: Any file type (text, PDF, docx, xlsx, CSV, JSON, code files)
  - **Write**: Create new files only, no modification of existing files
  - **Search**: Find files by name, content, or metadata
  - **Analysis**: Summarize documents, extract key information
- **Safety**: Sandboxed file operations, user confirmation for writes
- **Supported formats**: .txt, .md, .pdf, .docx, .xlsx, .csv, .json, .py, .js, etc.

### 4. Web Search Agent
- **API**: Linkup API
- **Usage policy**: ONLY for external information retrieval, NEVER send user data/files
- **Triggers**: Activated when local knowledge is insufficient
- **Privacy check**: Validate query doesn't contain PII before sending

### 5. Translation Agent
- **Capabilities**: Translate text between languages
- **Method**: Use local Ollama model (no external API for privacy)
- **Supported languages**: Major languages (EN, ES, FR, DE, ZH, JA, KO, etc.)

### 6. Code Assistant Agent
- **Option A**: Use Claude API (Sonnet 3.5 recommended for code quality)
- **Option B**: Use local model with code-specific fine-tuning
- **Capabilities**:
  - Code generation
  - Debugging and explanation
  - Refactoring suggestions
  - Documentation generation
- **Decision point**: Choose based on quality requirements vs. privacy concerns

### 7. Summarizer Agent
- **Purpose**: Meta-agent that summarizes outputs from other agents
- **Use cases**:
  - Long email threads → key points
  - Multiple calendar events → schedule overview
  - Web search results → concise summary
  - Document analysis → executive summary
- **Implementation**: Use local LLM with specialized summarization prompt

## UI/UX Requirements

### Main Interface
```
┌─────────────────────────────────────────────────────┐
│ [+] Tab 1    Tab 2    Tab 3                    [⚙] │
├─────────────────────────────────────────────────────┤
│                                                     │
│  User: Can you check my emails from Sarah?         │
│                                                     │
│  Assistant: [Email Agent activated]                │
│  Found 3 emails from Sarah:                        │
│  1. "Project Update" (2 hours ago)                 │
│  2. "Meeting Reschedule" (yesterday)               │
│  ...                                                │
│                                                     │
├─────────────────────────────────────────────────────┤
│ [Type your message...]                      [Send] │
└─────────────────────────────────────────────────────┘
```

### Required UI Elements
- Tab bar with add/close buttons
- Chat history display (scrollable)
- Input field with multi-line support
- Agent activity indicators ("🔍 Searching...", "📧 Reading emails...")
- Settings panel (API keys, model selection, preferences)
- Session history sidebar (optional, searchable)

## Development Best Practices

### Code Quality
- **Type hints**: Full type annotations throughout
- **Documentation**: Docstrings for all classes and functions
- **Comments**: Explain complex logic, not obvious code
- **Naming**: Clear, descriptive variable/function names
- **Structure**: Modular design with clear separation of concerns

### Error Handling
- Try-catch blocks around all external calls (API, file I/O)
- Graceful degradation (if one agent fails, others continue)
- User-friendly error messages
- Logging system for debugging (log to file, not console)

### Testing
- Unit tests for each agent
- Integration tests for agent orchestration
- Mock external APIs for testing
- Test edge cases (no internet, API failures, invalid inputs)

### Security & Privacy
- Never log sensitive data (passwords, tokens, email content)
- Encrypt stored credentials (use keyring library)
- Validate all user inputs
- Sandbox file operations
- Clear privacy policy about what data stays local vs. goes external

### Configuration
```python
# config.yaml
app:
  name: "AGI Desktop Assistant"
  version: "1.0.0"

llm:
  model: "llama3.1:8b"
  embedding_model: "nomic-embed-text"
  max_context_tokens: 6000
  temperature: 0.7

chromadb:
  persist_directory: "./data/chromadb"
  
agents:
  email:
    enabled: true
    mcp_server: "gmail-mcp"
  calendar:
    enabled: true
    mcp_server: "gcal-mcp"
  web_search:
    enabled: true
    api_key_env: "LINKUP_API_KEY"
```

## Project Structure
```
agi-desktop-assistant/
├── main.py                      # Entry point
├── config.yaml                  # Configuration
├── requirements.txt             # Dependencies
├── .env                         # API keys (gitignored)
│
├── gui/
│   ├── __init__.py
│   ├── app.py                   # Main Flet application
│   ├── components/
│   │   ├── chat_view.py        # Chat interface
│   │   ├── tab_manager.py      # Tab management
│   │   └── settings_panel.py   # Settings UI
│   └── styles.py               # UI theming
│
├── core/
│   ├── __init__.py
│   ├── orchestrator.py         # Main agentic loop
│   ├── planner.py              # Query planning
│   ├── context_manager.py      # Session + ChromaDB management
│   └── llm_client.py           # Ollama integration
│
├── agents/
│   ├── __init__.py
│   ├── base_agent.py           # Abstract base class
│   ├── email_agent.py
│   ├── calendar_agent.py
│   ├── file_agent.py
│   ├── web_search_agent.py
│   ├── translation_agent.py
│   ├── code_assistant_agent.py
│   └── summarizer_agent.py
│
├── utils/
│   ├── __init__.py
│   ├── logger.py               # Logging setup
│   ├── security.py             # Credential management
│   └── helpers.py              # Common utilities
│
├── tests/
│   ├── test_agents/
│   ├── test_core/
│   └── test_integration/
│
└── data/
    ├── chromadb/               # Vector store (gitignored)
    ├── sessions/               # Session logs (gitignored)
    └── credentials/            # OAuth tokens (gitignored)
```

## Development Phases

### Phase 1: Foundation (MVP)
1. Basic Flet GUI with single tab
2. Ollama integration + ChromaDB setup
3. Simple agentic loop (think-act-validate)
4. File handling agent (read-only)
5. Basic context management

### Phase 2: Core Agents
1. Email agent with Gmail integration
2. Calendar agent with Google Calendar
3. Web search agent with Linkup API
4. Tab management system

### Phase 3: Advanced Features
1. Translation agent
2. Code assistant agent
3. Summarizer agent
4. Cross-session reference capability
5. Advanced context compaction

### Phase 4: Polish
1. Comprehensive error handling
2. Full test coverage
3. UI/UX improvements
4. Documentation
5. Deployment packaging

## Acceptance Criteria
- ✅ All agents functional and testable
- ✅ Context maintained correctly within sessions
- ✅ No cross-session contamination
- ✅ Privacy requirements met (no leaks to external APIs)
- ✅ Graceful error handling throughout
- ✅ Code is well-documented and maintainable
- ✅ All unit and integration tests pass
- ✅ UI is responsive and intuitive

## Open Questions to Resolve
1. For code assistant: Claude API vs. local model? (Cost vs. privacy trade-off)
2. MCP server availability: Which agents have existing MCP servers?
3. File permissions: Should user approve each file access, or trust by directory?
4. Web search privacy: Should we parse/sanitize queries automatically or ask user first?