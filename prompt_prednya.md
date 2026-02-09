# Desktop Intelligence Agent - System Design Document

> An AGI-inspired multi-agent desktop assistant with persistent memory, session management, and parallel agent orchestration.

---

## Architecture Overview

```
+-----------------------------------------------------------------+
|                   Desktop App (Flet / Python)                   |
|          Flutter-rendered UI  |  Pure Python backend            |
+-----------------------------------------------------------------+
|                      Agent Orchestrator                         |
|  - Parallel dispatch  - Session router  - Priority queue        |
+-----------------------------------------------------------------+
|   Email  | Calendar | Files | Search | Code | Meeting | Memory  |
|   Agent  |  Agent   | Agent | Agent  | Agent|  Agent  | Agent   |
+-----------------------------------------------------------------+
|                     MCP Server Layer                            |
+-----------------------------------------------------------------+
|  Ollama (LLM + Embeddings)  |  ChromaDB/LanceDB  |  SQLite     |
+-----------------------------------------------------------------+
```

---

## 1. Email Agent

### Capabilities
- **Read** - Fetch, filter, and display emails (by sender, date, labels, unread status, attachments)
- **Write** - Compose and send emails with rich text, attachments, and CC/BCC
- **Summarize** - Generate concise summaries of long email threads
- **Draft Management** - Create, update, and delete drafts before sending
- **Style Matching** - Learn the user's writing tone and generate emails that match their style
- **Smart Reply** - Generate context-aware reply suggestions
- **Attachment Handling** - Download, preview, and attach files
- **Thread Analysis** - Extract action items, deadlines, and key decisions from threads
- **Spam/Priority Triage** - Auto-classify emails by urgency and importance

### MCP Servers
| Server | Package | Capabilities |
|--------|---------|-------------|
| **Google Workspace MCP** (Recommended) | `uvx workspace-mcp --tools gmail drive calendar tasks` | Full Gmail + Drive + Calendar + Tasks + Docs + Sheets; three tool tiers (Core/Extended/Complete) |
| mcp-gsuite | `MarkusPfundstein/mcp-gsuite` | Multi-account Gmail + Calendar + Tasks + Drive (read-only) + Sheets |
| Gmail MCP Server | `GongRzhe/Gmail-MCP-Server` | Send/receive with full attachment support; auto-auth for Claude Desktop |
| Gmail Style MCP | `PaulFidika/gmail-mcp-server` | Style-learning draft generation |
| **Microsoft MCP** (for Outlook) | `elyxlz/microsoft-mcp` | Outlook Mail + Calendar + OneDrive + Contacts via Graph API |
| MS 365 MCP | `softeria/ms-365-mcp-server` | List/get/send/delete mail + folders + calendar + files |
| Outlook MCP | `XenoXilus/outlook-mcp` | Email + Calendar + SharePoint via Graph API |

---

## 2. Calendar Agent

### Capabilities
- **Daily Briefing** - Remind the user every morning of all meetings/events for the day
- **Event Extraction** - Automatically detect meeting invitations from emails and add to calendar
- **Create/Update/Delete Events** - Full CRUD on calendar entries with natural language
- **Recurring Events** - Handle daily/weekly/monthly patterns
- **Free/Busy Queries** - Check availability across multiple calendars
- **Smart Scheduling** - Find optimal meeting slots across participants
- **Conflict Detection** - Alert when new events overlap existing ones
- **Meeting Prep** - Pull relevant context (emails, notes, files) before a meeting starts
- **Cross-Platform Sync** - Work with both Google Calendar and Outlook Calendar

### MCP Servers
| Server | Package | Capabilities |
|--------|---------|-------------|
| **Google Calendar MCP** (Recommended) | `nspady/google-calendar-mcp` | Multi-account/multi-calendar; create/update/delete; recurring events; free/busy queries; natural language scheduling |
| Google Workspace MCP | `taylorwilsdon/google_workspace_mcp` | Calendar as part of full Google Workspace suite |
| **Microsoft MCP** (for Outlook) | `elyxlz/microsoft-mcp` | Calendar via Microsoft Graph API |
| Outlook Calendar MCP | `merajmehrabi/outlook-calendar` | Event management + find available slots + .ics file support |
| Cal MCP | `sms03/cal-mcp` | Cross-provider calendar management + availability finder |

---

## 3. File Handler Agent

### Capabilities
- **Browse** - List and navigate directory structures
- **Read/Write** - Open, read, create, and modify files (with user confirmation every time)
- **Search** - Find files by name, type, content, or metadata
- **Move/Organize** - Rename, move, copy files across directories
- **Metadata** - Get file size, type, creation date, last modified
- **Permission Gating** - Always prompt the user before any file operation (read/write/delete)
- **Smart Organization** - Suggest file organization based on content and usage patterns
- **Git Integration** - Track changes, diff, commit, branch operations on code repositories
- **Directory Access Control** - Configurable allowed directories to prevent unauthorized access

### MCP Servers
| Server | Package | Capabilities |
|--------|---------|-------------|
| **Filesystem MCP** (Official, Recommended) | `npx @modelcontextprotocol/server-filesystem /allowed/dir` | Read/write files; create/list dirs; move; search; metadata; directory-level access control |
| Git MCP | `@modelcontextprotocol/server-git` | Git operations: diff, log, commit, branch, search repos |

### Security Note
- Filesystem MCP versions prior to 2025.7.1 had path traversal vulnerabilities (CVE-2025-53109, CVE-2025-53110). Use latest version.
- All file operations must go through a confirmation prompt before execution.

---

## 4. Web Search Agent

### Capabilities
- **Search** - Natural language web queries with real-time results
- **Deep Search** - Control search depth, topic, and time range filters
- **Content Extraction** - Fetch and parse full page content from URLs
- **Domain Filtering** - Include/exclude specific domains from results
- **Site Mapping** - Crawl and map website structure
- **News Search** - Dedicated news, image, and video search
- **Direct Q&A** - Get AI-summarized answers directly from search results
- **Local Search** - Business/POI lookup with location context
- **Privacy-Focused Option** - Use Brave for independent index without tracking

### MCP Servers
| Server | Package | Capabilities |
|--------|---------|-------------|
| **LinkUp MCP** (Recommended per original spec) | `LinkupPlatform/linkup-mcp-server` | Web search + page content fetching; curated/premium sources; local or hosted endpoint |
| **Tavily MCP** (Most feature-rich) | `tavily-ai/tavily-mcp` | search, extract, map, crawl tools; depth control; domain filtering; images; direct QA; 1,000 free calls/month |
| Brave Search MCP | `brave/brave-search-mcp-server` | Web/local/image/video/news search; AI summarization; privacy-focused; free tier |
| Web Search MCP (Local) | `mrkrsl/web-search-mcp` | Simple locally-hosted search for local LLMs |

---

## 5. Code Assistant Agent

### Capabilities
- **Code Generation** - Write code from natural language descriptions using Claude
- **Code Editing** - Modify existing files with context-aware changes
- **Debugging** - Identify and fix bugs with explanation
- **Code Review** - Analyze code quality, suggest improvements
- **Refactoring** - Restructure code while preserving behavior
- **Git Workflow** - Create branches, commit, open PRs autonomously
- **Multi-File Operations** - Coordinate changes across multiple files
- **Browser Testing** - Automate browser interactions to verify changes
- **Error Monitoring** - Pull and analyze production errors from Sentry
- **Sequential Reasoning** - Break complex problems into structured, trackable steps
- **Agents-in-Agents** - Spawn sub-agents for specialized tasks within code workflows

### MCP Servers
| Server | Package | Capabilities |
|--------|---------|-------------|
| **Claude Code MCP** (Recommended) | `steipete/claude-code-mcp` | Claude Code as one-shot sub-agent; file CRUD; git operations; complex multi-step workflows; configurable timeout |
| GitHub MCP | `@modelcontextprotocol/server-github` | Browse/search repos; create/update issues + PRs; execute and commit code |
| **Playwright MCP** | `microsoft/playwright-mcp` | Browser automation for testing; click/type/wait/screenshot; accessibility snapshots; token-efficient CLI mode |
| Sentry MCP | `getsentry/sentry-mcp` | Access production errors, issues, projects; AI-powered root cause analysis |
| **Sequential Thinking MCP** | `npx @modelcontextprotocol/server-sequential-thinking` | Structured step-by-step reasoning; revision and branching; 54% improvement in sequential decision-making |

---

## 6. Meeting Notes Agent

### Capabilities
- **Recording** - Send bots to Zoom, Google Meet, and Microsoft Teams to record
- **Transcription** - Automatic speech-to-text with speaker identification
- **Summarization** - Generate concise meeting summaries (bullet points or paragraphs)
- **Key Moments** - AI-powered detection of key decisions, action items, and deadlines
- **Search Across Meetings** - Full-text and semantic search across all past transcripts
- **Pattern Analysis** - Identify recurring themes and topics across multiple meetings
- **Participant Tracking** - Filter meetings by attendees
- **Action Item Extraction** - Pull out TODOs and assign ownership
- **Meeting Prep Context** - Pull relevant past meeting notes before new meetings
- **Cross-Platform** - Support Zoom, Google Meet, and Microsoft Teams

### MCP Servers
| Server | Package | Capabilities |
|--------|---------|-------------|
| **tl;dv MCP** (Recommended) | `tldv-public/tldv-mcp-server` | Zoom + Meet + Teams; list/filter/search meetings; transcripts; AI highlights/key moments; supports agent patterns (competitor intel, performance review, onboarding) |
| Meeting-BaaS MCP | `Meeting-Baas/meeting-mcp` | Bot creation for Zoom/Meet/Teams; recording + transcription; `findKeyMoments`; calendar OAuth; QR bot avatars |
| Fireflies MCP | `npx @props-labs/mcp/fireflies` | Full transcript details with speakers; keyword search; summary generation; transforms hours of review into minutes |
| Otter.ai MCP | Otter.ai Marketplace | Search all transcripts; cross-meeting pattern analysis; content generation from meeting data; enterprise OAuth |
| MeetGeek MCP | MeetGeek Platform | AI-powered meeting data transformation |

---

## 7. Memory Agent (Context & Session Management)

### Capabilities
- **Session Isolation** - Each tab/session maintains its own conversation context
- **Cross-Session Recall** - Query and retrieve context from any previous session by reference
- **Persistent Knowledge Graph** - Store entities, relationships, and observations that survive restarts
- **Semantic Search** - Find relevant memories by meaning, not just keywords
- **Automatic Context Injection** - Relevant memories auto-loaded into agent context
- **Preference Learning** - Remember user preferences, habits, and patterns
- **Decision History** - Track past decisions with rationale for future reference
- **Knowledge Graph Visualization** - Interactive D3.js graph of stored knowledge
- **Cross-Agent Memory** - Any agent can store/retrieve from shared memory layer

### MCP Servers
| Server | Package | Capabilities |
|--------|---------|-------------|
| **Knowledge Graph Memory** (Official, Recommended) | `@modelcontextprotocol/server-memory` | Entity/relation CRUD; observations; directed graph; persists across sessions |
| **mcp-memory-service** (Recommended for semantic) | `doobidoo/mcp-memory-service` | ChromaDB + sentence transformers; 5ms retrieval; D3.js visualization; web dashboard; optional cloud sync |
| Basic Memory | `basicmachines-co/basic-memory` | Markdown-based semantic graph; local-first; bidirectional; works with Obsidian; SQLite index |
| **Mem0 / OpenMemory** | `mem0ai/mem0` | Universal memory layer across all MCP tools; auto-capture; store in one tool, retrieve in another |
| Memory Keeper | `mkreyman/mcp-memory-keeper` | Persistent context for coding assistants; preserves work history and decisions |

---

## Session & Tab Management

### Architecture
```
Session Manager
  |
  +-- Session 1: "Project Alpha Planning"
  |     +-- Conversation history (messages[])
  |     +-- Active agents: [Email, Calendar]
  |     +-- Session-specific memory context
  |     +-- Embeddings index (session-scoped)
  |
  +-- Session 2: "Code Review Sprint"
  |     +-- Conversation history (messages[])
  |     +-- Active agents: [Code, Git]
  |     +-- Session-specific memory context
  |     +-- Embeddings index (session-scoped)
  |
  +-- Cross-Session Memory (shared knowledge graph + vector store)
```

### How It Works
- Each **tab** = isolated session with its own conversation history and agent state
- Sessions are persisted to disk (SQLite + vector store) for cross-restart survival
- Cross-session queries: "What did we discuss in Session 1?" triggers a semantic search across that session's embedded conversation history
- Memory Agent maintains both **session-scoped** and **global** memory layers

---

## Parallel Agent Orchestration

### How It Works
- When a user prompt requires multiple agents, the **Orchestrator** decomposes it into sub-tasks
- Sub-tasks are dispatched to agents **in parallel** when independent
- Results are aggregated and presented as a unified response
- Dependency-aware: if Agent B needs output from Agent A, the orchestrator sequences them

### Example
```
User: "Summarize my unread emails, check my calendar for tomorrow,
       and find any meeting notes from last week's standup"

Orchestrator detects 3 independent tasks:
  -> Email Agent:   Summarize unread emails
  -> Calendar Agent: Get tomorrow's events
  -> Meeting Agent:  Search transcripts for "standup" from last week

All three run in parallel. Results merged into single response.
```

---

## Tech Stack

### Cross-Platform Desktop Framework
| Option | Language | Why |
|--------|----------|-----|
| **Flet** (Selected) | Pure Python | Flutter-rendered native UI from Python; async-native; single process; compiles to Win/Mac/Linux/iOS/Android/Web via `flet build`; Material Design 3 components; ~20-50MB bundle; no JavaScript or Rust required |

### LLM & Embeddings (Local via Ollama)
| Component | Model | Details |
|-----------|-------|---------|
| **LLM (Primary)** | Claude API (Anthropic) | Main reasoning engine for all agents |
| **LLM (Local/Fallback)** | Ollama + `llama3.2` or `mistral` | Offline capability; privacy-sensitive tasks; reduced API costs |
| **Embeddings** | Ollama + `nomic-embed-text` | 1,024 dims; 8,192 token context; ~0.5GB; surpasses OpenAI ada-002 |
| **Embeddings (Heavy)** | Ollama + `mxbai-embed-large` | 1,024 dims; ~1.2GB; SOTA for BERT-large class; outperforms OpenAI text-embedding-3-large |
| **Embeddings (Lightweight)** | Ollama + `all-minilm` | 384 dims; ~0.1GB; great for limited hardware |
| **Embeddings (Multilingual)** | Ollama + `qwen3-embedding` | Up to 8B params; #1 on MTEB multilingual leaderboard |

### Ollama Setup
```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh   # Linux/Mac
# Windows: download from https://ollama.com/download

# Pull models
ollama pull llama3.2          # Local LLM
ollama pull nomic-embed-text  # Embeddings (recommended default)
ollama pull mxbai-embed-large # Embeddings (high accuracy)

# Ollama runs a local API at http://localhost:11434
# Embedding endpoint: POST http://localhost:11434/api/embeddings
```

### Vector Database & Storage
| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Vector Store** (Recommended) | ChromaDB | Embedded vector DB; Python/JS SDKs; integrates with sentence-transformers; has its own MCP server (`chroma-core/chroma-mcp`) |
| Vector Store (Alternative) | LanceDB | Apache Arrow format; memory-mapped disk; scales to petabytes; zero config |
| Vector Store (Lightweight) | sqlite-vec | Adds vector search to existing SQLite; ultra-lightweight |
| **Relational Store** | SQLite | Session metadata, conversation history, user preferences, agent state |
| **Knowledge Graph** | MCP Memory Server | Entity-relation graph for persistent agent memory |

### Memory & Context Storage Architecture
```
+---------------------------------------------------+
|              Memory & Context Layer                |
+---------------------------------------------------+
|                                                   |
|  SQLite                                           |
|  - Sessions table (id, name, created_at)          |
|  - Messages table (session_id, role, content, ts) |
|  - Agent state (agent_id, session_id, state_json) |
|  - User preferences (key, value)                  |
|                                                   |
|  ChromaDB (Vector Store)                          |
|  - Collection per session (session embeddings)    |
|  - Global collection (cross-session memories)     |
|  - Email embeddings (for semantic email search)   |
|  - Meeting transcript embeddings                  |
|  - Embedding model: nomic-embed-text via Ollama   |
|                                                   |
|  Knowledge Graph (MCP Memory Server)              |
|  - Entities: people, projects, decisions, dates   |
|  - Relations: "works_on", "decided", "owns"       |
|  - Observations: discrete facts per entity        |
|                                                   |
+---------------------------------------------------+
```

### How Embeddings Are Used
1. **Conversation Memory** - Each message is embedded and stored per-session in ChromaDB. When a user references a past session, semantic search retrieves the most relevant messages.
2. **Email Search** - Email bodies are embedded for semantic search ("find that email about the budget from last month").
3. **Meeting Notes** - Transcripts are chunked and embedded for fast retrieval ("what did John say about the API redesign?").
4. **Cross-Session Recall** - Global memory collection enables "what did we discuss last week about Project Alpha?" across all sessions.
5. **RAG Pipeline** - Retrieved chunks are injected into the LLM prompt as context for grounded, accurate responses.

---

## Additional Agents & Capabilities

### 8. Browser Automation Agent
- Powered by **Playwright MCP** (`microsoft/playwright-mcp`)
- Fill web forms, extract data from pages, take screenshots
- Verify deployed changes in real-time
- Accessibility-aware (snapshot mode for screen readers)

### 9. Workflow/Integration Agent
- **Zapier MCP** - Connect to 7,000+ apps via natural language
- **n8n MCP** - Trigger complex automation workflows
- **Composio** - Universal gateway replacing 22+ individual MCP servers; 500+ integrations; SOC2/ISO certified

### 10. Knowledge Base Agent
- **Notion MCP** - Read/write Notion pages, databases, tasks as real-time context
- **Basic Memory** - Bidirectional markdown knowledge base compatible with Obsidian

### 11. Reasoning/Planning Agent
- **Sequential Thinking MCP** (`@modelcontextprotocol/server-sequential-thinking`)
- Breaks complex problems into trackable steps with revision and branching
- 54% improvement in sequential decision-making tasks
- Powers the orchestrator's task decomposition

---

## Full MCP Configuration (claude_desktop_config.json)

```json
{
  "mcpServers": {
    "gmail": {
      "command": "uvx",
      "args": ["workspace-mcp", "--tools", "gmail", "drive", "calendar", "tasks"]
    },
    "outlook": {
      "command": "npx",
      "args": ["-y", "microsoft-mcp"],
      "env": { "MS_CLIENT_ID": "...", "MS_CLIENT_SECRET": "..." }
    },
    "google-calendar": {
      "command": "npx",
      "args": ["-y", "google-calendar-mcp"],
      "env": { "GOOGLE_OAUTH_CREDENTIALS": "..." }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/Documents"]
    },
    "git": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-git"]
    },
    "linkup-search": {
      "command": "npx",
      "args": ["-y", "linkup-mcp-server"],
      "env": { "LINKUP_API_KEY": "..." }
    },
    "tavily-search": {
      "command": "npx",
      "args": ["-y", "tavily-mcp"],
      "env": { "TAVILY_API_KEY": "..." }
    },
    "code-assistant": {
      "command": "npx",
      "args": ["-y", "claude-code-mcp"],
      "env": { "ANTHROPIC_API_KEY": "..." }
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "..." }
    },
    "playwright": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-playwright"]
    },
    "meeting-notes": {
      "command": "npx",
      "args": ["-y", "tldv-mcp-server"],
      "env": { "TLDV_API_KEY": "..." }
    },
    "fireflies": {
      "command": "npx",
      "args": ["-y", "@props-labs/mcp/fireflies"],
      "env": { "FIREFLIES_API_KEY": "..." }
    },
    "memory": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"]
    },
    "memory-semantic": {
      "command": "npx",
      "args": ["-y", "mcp-memory-service"],
      "env": { "EMBEDDING_MODEL": "nomic-embed-text", "OLLAMA_URL": "http://localhost:11434" }
    },
    "chromadb": {
      "command": "npx",
      "args": ["-y", "chroma-mcp"],
      "env": { "CHROMA_HOST": "localhost", "CHROMA_PORT": "8000" }
    },
    "sequential-thinking": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-sequential-thinking"]
    }
  }
}
```

---

## Tech Requirements Summary

| Category | Technology | Purpose |
|----------|-----------|---------|
| **Desktop Framework** | Flet (Pure Python) | Cross-platform app — Win/Mac/Linux/iOS/Android/Web |
| **Language** | Python 3.11+ | Entire application — UI, agents, orchestration, memory |
| **Primary LLM** | Claude API (Anthropic) | Main reasoning engine |
| **Local LLM** | Ollama (`llama3.2` / `mistral`) | Offline fallback, privacy tasks, cost reduction |
| **Embeddings** | Ollama (`nomic-embed-text`) | Vector representations for semantic search |
| **Vector Database** | ChromaDB | Semantic memory storage and retrieval |
| **Relational Database** | SQLite (`aiosqlite`) | Sessions, messages, preferences, agent state |
| **Knowledge Graph** | MCP Memory Server | Persistent entity-relation memory |
| **Agent Protocol** | Model Context Protocol (MCP) | Standardized agent-tool communication |
| **Auth (Google)** | OAuth 2.0 | Gmail, Calendar, Meet, Drive |
| **Auth (Microsoft)** | Azure AD + Graph API | Outlook, Teams, OneDrive |
| **Browser Automation** | Playwright MCP | Testing, web scraping, form filling |
| **Search APIs** | LinkUp + Tavily | Web search with content extraction |
| **Meeting Transcription** | tl;dv / Fireflies / Otter.ai | Recording + transcription + summarization |
| **Workflow Automation** | Zapier MCP / n8n | 7,000+ app integrations |
| **Package Manager** | uv (Python) + npm (MCP servers) | Dependency management |

---

## Key Design Principles

1. **User Confirmation for Sensitive Actions** - File operations, email sends, and calendar changes always require explicit user approval
2. **Session Isolation with Cross-Session Intelligence** - Each tab has its own context, but the memory layer connects them
3. **Parallel by Default** - Independent agent tasks run concurrently; dependent tasks are sequenced automatically
4. **Local-First** - Ollama for LLM/embeddings, ChromaDB for vectors, SQLite for state - everything works offline
5. **Privacy-Aware** - Sensitive data stays local; cloud APIs are opt-in per agent
6. **MCP-Native** - All agent capabilities exposed through standardized MCP servers for interoperability
