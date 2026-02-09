# ATLAS: Adaptive Task-Learning Agent System
## Implementation Plan for AGI-Inspired Desktop Intelligence Agent

---

## Context

College project competing against other teams to build the best AGI-inspired desktop intelligence agent. Must demonstrate multi-domain reasoning, contextual memory, autonomous planning, Linkup API integration, and privacy-first local architecture.

**Team:** 3-4 expert Python/AI engineers, 4-6 weeks
**Target:** macOS, Flet UI, pure Python stack
**LLM:** Local-first (Ollama/Qwen3 8B) + opt-in cloud (Claude/GPT)
**Integrations:** Gmail, Google Calendar, Linkup, Filesystem, WhatsApp, macOS Automation, Translation
**Key Differentiator:** Master-Agent architecture with MCP tool backbone + session-based UI

---

## Architecture Overview

```
+========================================================================+
|                          ATLAS Desktop Agent                           |
+========================================================================+
|                                                                        |
|   FLET UI (Tabbed Sessions)                                           |
|   +----+ +----+ +----+ +----+                                         |
|   |Ses1| |Ses2| |Ses3| | +  | <-- Each tab = independent session     |
|   +----+ +----+ +----+ +----+                                         |
|   +---------------------------+  +-------------------------------+     |
|   | Reasoning Trace Panel     |  | Chat Panel + File Picker     |     |
|   | (Live step visualization) |  | (Messages + Tool Cards)      |     |
|   +---------------------------+  +-------------------------------+     |
|                          |                                             |
|           page.run_task() (Flet async bridge)                         |
|                          |                                             |
|   MASTER ORCHESTRATOR (LangGraph StateGraph)                          |
|   - Understands user intent via LLM                                   |
|   - Plans which specialist agents to invoke                           |
|   - Coordinates multi-agent workflows                                 |
|   - Human-in-the-loop confirmation for destructive actions            |
|                          |                                             |
|   SPECIALIST AGENTS (7 agents)                                        |
|   +-------+ +--------+ +------+ +--------+ +------+ +-----+ +-----+ |
|   | Email | |Calendar| | File | |Research | |Transl| |Whats| |macOS| |
|   | Agent | | Agent  | |Handler| | Agent  | |Agent | |App  | |Auto | |
|   | [LLM] | [simple] |[simple]| | [LLM]  | |[LLM] |[simp]|[simp]| |
|   +-------+ +--------+ +------+ +--------+ +------+ +-----+ +-----+ |
|       |         |          |         |         |        |       |     |
|       v         v          v         v         v        v       v     |
|   AGENT-OWNED MCP SERVERS        SHARED MCP POOL                     |
|   +------+ +------+             +--------+ +------+                  |
|   |Gmail | |GCal  |             | Linkup | | File |                  |
|   | MCP  | | MCP  |             |  MCP   | | Sys  |                  |
|   +------+ +------+             +--------+ +------+                  |
|   +--------+ +------+                                                |
|   |WhatsApp| |macOS |                                                |
|   |  MCP   | |Auto  |                                                |
|   +--------+ +------+                                                |
|                          |                                             |
|   MEMORY SUBSYSTEM                                                    |
|   +----------------+ +---------------+ +-------------------+         |
|   | Session Memory | | Long-term     | | Episodic          |         |
|   | (per-tab       | | (ChromaDB)    | | (Pattern Store)   |         |
|   |  SQLite)       | | (shared)      | | (cross-session)   |         |
|   +----------------+ +---------------+ +-------------------+         |
|                          |                                             |
|   LLM ABSTRACTION                                                     |
|   +-------------------+  +---------------------+                     |
|   | Local: Ollama     |  | Cloud: Claude/GPT   |                     |
|   | (Qwen3 8B default)|  | (Opt-in with consent)|                    |
|   +-------------------+  +---------------------+                     |
+========================================================================+
```

---

## Why This Wins (8 Key Differentiators)

1. **Master → Specialist Agent hierarchy** — Clean delegation model mirrors real organizations. Master plans, specialists execute. Judges see genuine autonomy.
2. **Session/Tab architecture** — Each task is a separate, persistent session. Cross-session context via shared memory. Feels like a real desktop app, not a chatbot.
3. **MCP-native tool backbone** — Specialist agents own MCP servers. Adding a new capability = new YAML config + new agent wrapper. Infinite extensibility.
4. **Visible reasoning trace** — Panel shows Master's plan, agent delegation, tool calls, and evaluation in real-time. Directly proves "Reasoning Quality".
5. **Privacy as a visible feature** — Query sanitization in trace, confirmation dialogs, local-first LLM. Most teams will ignore this.
6. **Episodic memory + knowledge transfer** — Agent reuses strategies across sessions and domains. Closest to AGI-like behavior.
7. **Deep Linkup integration** — Smart query formulation, privacy filtering, multi-query chaining, source synthesis.
8. **7 specialist agents** — Email, Calendar, File Handler, Research, Translator, WhatsApp, macOS Automation. Broadest domain coverage.

---

## Core Architecture: Master-Agent Model

### How It Works

```
User Input: "Summarize my unread emails, extract deadlines, add them to calendar"
         |
         v
   MASTER ORCHESTRATOR (LLM-powered)
   1. Parses intent
   2. Checks session memory + cross-session memory
   3. Creates execution plan:
      Step 1: EmailAgent.read_unread() -> get emails
      Step 2: EmailAgent.summarize(emails) -> summaries + deadlines [LLM]
      Step 3: ResearchAgent.research(unknown_entities) -> context [LLM]
      Step 4: CalendarAgent.create_events(deadlines) -> confirmation
   4. Dispatches to specialist agents in order
   5. Collects results, evaluates quality
   6. Compiles final response for user
         |
         v
   SPECIALIST AGENTS execute their tools:
   - EmailAgent calls Gmail MCP: gmail_search, gmail_get_message
   - EmailAgent uses LLM to summarize (hybrid agent)
   - ResearchAgent calls Linkup MCP: linkup_search (hybrid agent)
   - CalendarAgent calls GCal MCP: calendar_create_event (simple agent)
         |
         v
   Results compiled -> displayed in Chat Panel + Reasoning Trace
```

### Agent Classification

| Agent | Type | Has Own LLM? | Owned MCP Servers | Shared Tools |
|-------|------|-------------|-------------------|--------------|
| **Email Agent** | Hybrid | Yes (summarize, draft, analyze) | Gmail MCP | Filesystem, Linkup |
| **Calendar Agent** | Simple | No (structured CRUD) | Google Calendar MCP | — |
| **File Handler Agent** | Simple | No (parse + extract) | — | Filesystem MCP |
| **Research Agent** | Hybrid | Yes (query formulation, synthesis) | — | Linkup MCP |
| **Translator Agent** | Hybrid | Yes (translation reasoning) | — | — |
| **WhatsApp Agent** | Simple | No (send/read messages) | WhatsApp MCP | — |
| **macOS Automation Agent** | Simple | No (execute system commands) | macOS Automation MCP | — |

**Hybrid agents** get their own LLM calls for nuanced tasks (summarization, research, translation).
**Simple agents** are tool-execution wrappers — the Master tells them exactly what to do.

---

## Session/Tab Architecture

### Session Model

Each tab in the Flet UI represents an independent **session**:

```python
@dataclass
class Session:
    id: str                          # UUID
    title: str                       # User-editable or auto-generated
    created_at: datetime
    updated_at: datetime
    status: str                      # "active", "completed", "archived"
    messages: list[Message]          # Conversation history
    context: dict                    # Active entity references
    artifacts: list[Artifact]        # Files produced, events created, etc.
    summary_embedding_id: str        # ChromaDB ID for cross-session search
```

### Cross-Session Context (Auto + Manual Hybrid)

**Auto-indexing:** When a session completes a meaningful action, its results are automatically embedded and stored in the shared ChromaDB:
- Session summary (auto-generated)
- Entities mentioned (people, companies, documents)
- Actions taken (emails sent, events created, files processed)
- Artifacts produced (summaries, drafts, extractions)

**Auto-retrieval:** When the Master plans a new task, it searches past sessions for relevant context:
```
Master receives: "Draft a reply to TechNova about the partnership"
  -> Searches ChromaDB: "TechNova partnership"
  -> Finds: Session #3 researched TechNova (company profile, key contacts)
  -> Injects that context into the current plan
```

**Manual linking:** User can explicitly reference another session:
- "Use the research from tab 2"
- "Apply the same approach as my last email task"

---

## Specialist Agent Implementations

### Base Agent Interface

```python
# atlas/agents/base.py

from abc import ABC, abstractmethod
from typing import Any, Optional
from langchain_core.tools import BaseTool

class SpecialistAgent(ABC):
    """Base class for all specialist agents."""

    name: str                        # "email", "calendar", etc.
    description: str                 # For Master's planning context
    owned_mcp_tools: list[BaseTool]  # Tools this agent owns
    shared_tools: list[BaseTool]     # Shared pool tools it can access
    has_llm: bool                    # Whether this agent uses LLM

    @abstractmethod
    async def execute(self, task: AgentTask) -> AgentResult:
        """Execute a task delegated by the Master."""
        pass

    def get_capabilities(self) -> list[str]:
        """Return list of capability descriptions for Master's planning."""
        pass
```

### Email Agent (Hybrid — has LLM)

```python
class EmailAgent(SpecialistAgent):
    name = "email"
    description = "Read, search, summarize, and draft emails via Gmail"
    has_llm = True

    # Owned MCP: Gmail MCP
    # Shared: Filesystem MCP (for attachments), Linkup MCP (for research)

    capabilities = [
        "read_unread_emails",      # Gmail MCP: gmail_search + gmail_get
        "search_emails(query)",     # Gmail MCP: gmail_search
        "summarize_email(id)",      # LLM: summarize email content
        "extract_deadlines(id)",    # LLM: find dates and deadlines
        "draft_reply(id, context)", # LLM: compose reply with context
        "send_email(to, subject, body)",  # Gmail MCP: gmail_send
        "get_attachments(id)",      # Gmail MCP: gmail_get_attachment
    ]
```

### Calendar Agent (Simple — no LLM)

```python
class CalendarAgent(SpecialistAgent):
    name = "calendar"
    description = "Create, read, update, and delete Google Calendar events"
    has_llm = False

    # Owned MCP: Google Calendar MCP

    capabilities = [
        "list_events(date_range)",
        "create_event(title, start, end, description, attendees)",
        "update_event(id, changes)",
        "delete_event(id)",
        "find_free_slots(date, duration)",
    ]
```

### Research Agent (Hybrid — has LLM)

```python
class ResearchAgent(SpecialistAgent):
    name = "research"
    description = "Web research via Linkup: entity lookup, fact-checking, news, person research"
    has_llm = True

    # Shared: Linkup MCP

    capabilities = [
        "research_entity(name)",           # Multi-query: overview + news + people
        "verify_claim(claim, context)",     # Deep search for fact-checking
        "research_person(name, company)",   # Professional background lookup
        "get_latest_news(topic)",           # Temporal search
        "fetch_webpage(url)",              # Linkup fetch for specific URLs
    ]

    # Key: Privacy filter strips PII before any Linkup query
    # Key: Smart query formulation (not pass-through)
    # Key: Multi-query chaining for comprehensive research
    # Key: Source synthesis with confidence scoring
```

### File Handler Agent (Simple — no LLM)

```python
class FileHandlerAgent(SpecialistAgent):
    name = "file_handler"
    description = "Read, process, and extract content from PDF, DOCX, and images"
    has_llm = False

    # Shared: Filesystem MCP
    # Internal tools: PyMuPDF, python-docx, pytesseract

    capabilities = [
        "read_pdf(path) -> text",
        "read_docx(path) -> text",
        "ocr_image(path) -> text",
        "list_files(directory, pattern)",
        "write_file(path, content)",
        "search_files(query)",
    ]
```

### Translator Agent (Hybrid — has LLM)

```python
class TranslatorAgent(SpecialistAgent):
    name = "translator"
    description = "Translate text between languages, detect language, localize content"
    has_llm = True

    capabilities = [
        "translate(text, source_lang, target_lang)",
        "detect_language(text)",
        "translate_document(path, target_lang)",  # Uses FileHandler internally
        "localize_email_draft(draft, target_lang, cultural_context)",
    ]
```

### WhatsApp Agent (Simple — no LLM)

```python
class WhatsAppAgent(SpecialistAgent):
    name = "whatsapp"
    description = "Send and read WhatsApp messages to individuals and groups"
    has_llm = False

    # Owned MCP: WhatsApp MCP

    capabilities = [
        "send_message(contact, text)",
        "send_group_message(group, text)",
        "read_recent_messages(contact, count)",
        "search_messages(query)",
    ]
```

### macOS Automation Agent (Simple — no LLM)

```python
class MacOSAgent(SpecialistAgent):
    name = "macos"
    description = "macOS desktop automation: open apps, window management, screenshots, system control"
    has_llm = False

    # Owned MCP: automation-mcp

    capabilities = [
        "open_application(name)",
        "take_screenshot(region?)",
        "type_text(text)",
        "press_keys(keys)",
        "get_active_window()",
        "list_windows()",
        "move_window(app, position)",
    ]
```

---

## Master Orchestrator (LangGraph)

### State Definition

```python
class MasterState(TypedDict):
    # Session info
    session_id: str
    messages: Annotated[Sequence[BaseMessage], operator.add]
    user_input: str

    # Planning
    plan: Optional[dict]              # Structured plan from Master
    agent_tasks: list[AgentTask]      # Tasks delegated to specialists
    current_task_index: int

    # Results
    agent_results: list[AgentResult]  # Results from specialist agents

    # Evaluation
    evaluation: Optional[dict]
    retry_count: int

    # Memory
    session_memory: dict              # Current session context
    cross_session_context: list[dict] # Relevant past session data

    # Output
    final_response: str
    reasoning_trace: list[dict]       # For UI panel
    artifacts: list[dict]             # Files, events, etc. created
```

### Graph Flow

```
Entry
  |
  v
[retrieve_memory] -- Pull session context + search past sessions
  |
  v
[plan] -- Master LLM analyzes intent, creates agent_tasks list
  |        Each task specifies: agent_name, capability, args, needs_confirmation
  v
[route_tasks] -- Conditional edge:
  |              - If any task needs_research=True -> [research] first
  |              - If tasks need user confirmation -> [confirm]
  |              - Otherwise -> [execute]
  v
[research] -- Research Agent gathers info via Linkup
  |            Results injected into subsequent tasks' context
  v
[confirm] -- Human-in-the-loop for destructive actions
  |           (send email, create event, send WhatsApp)
  |           User approves/modifies/rejects in UI
  v
[execute] -- Dispatch tasks to specialist agents sequentially/parallel
  |           Simple agents: direct MCP tool calls
  |           Hybrid agents: LLM + MCP tool calls
  v
[evaluate] -- Master LLM checks: completeness, correctness, quality
  |            If failed + retries left -> back to [execute]
  |            If passed -> [respond]
  v
[respond] -- Compile final response, store session results, update memory
  |
  v
END
```

### Master's Planning Prompt

```
You are the Master Orchestrator for ATLAS. You have these specialist agents:

{agent_capabilities_summary}

For the user's request, create a plan:
1. Identify which agents are needed
2. Order tasks by dependencies (parallel where possible)
3. Mark tasks needing research (unknown entities, claims to verify)
4. Mark tasks needing user confirmation (sending messages, creating events)
5. If memory context includes relevant past sessions, reference them

Output a JSON plan:
{
  "intent": "brief description of user's goal",
  "tasks": [
    {
      "agent": "email",
      "capability": "read_unread_emails",
      "args": {},
      "depends_on": [],
      "needs_research": false,
      "needs_confirmation": false,
      "reason": "why this step is needed"
    }
  ]
}
```

---

## MCP Integration Layer

### Agent-Owned + Shared Pool Model

```yaml
# config/mcp_servers.yaml

# AGENT-OWNED: Only accessible by their designated agent
agent_owned:
  gmail:
    owner: "email"
    command: "uvx"
    args: ["workspace-mcp"]
    transport: "stdio"
    env:
      GOOGLE_OAUTH_CLIENT_ID: "${GOOGLE_OAUTH_CLIENT_ID}"
      GOOGLE_OAUTH_CLIENT_SECRET: "${GOOGLE_OAUTH_CLIENT_SECRET}"

  google_calendar:
    owner: "calendar"
    command: "uvx"
    args: ["workspace-mcp"]
    transport: "stdio"
    env:
      GOOGLE_OAUTH_CLIENT_ID: "${GOOGLE_OAUTH_CLIENT_ID}"
      GOOGLE_OAUTH_CLIENT_SECRET: "${GOOGLE_OAUTH_CLIENT_SECRET}"

  whatsapp:
    owner: "whatsapp"
    command: "uvx"
    args: ["whatsapp-mcp"]
    transport: "stdio"

  macos_automation:
    owner: "macos"
    command: "npx"
    args: ["-y", "automation-mcp"]
    transport: "stdio"

# SHARED POOL: Accessible by any agent that needs them
shared:
  linkup:
    url: "https://mcp.linkup.so/mcp"
    transport: "http"
    env:
      apiKey: "${LINKUP_API_KEY}"

  filesystem:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "${HOME}/Documents"]
    transport: "stdio"
```

### MCPManager

```python
class MCPManager:
    """Manages MCP server connections with agent-owned + shared pool model."""

    async def initialize(self):
        """Start all MCP servers, build tool registry."""
        # Uses langchain-mcp-adapters MultiServerMCPClient
        # Auto-discovers tools from each server
        # Builds ownership map: agent_name -> [tools]
        # Builds shared pool: [tools accessible to all]

    def get_tools_for_agent(self, agent_name: str) -> list[BaseTool]:
        """Return owned tools + shared pool tools for a specific agent."""
        owned = self.agent_tools.get(agent_name, [])
        shared = self.shared_tools
        return owned + shared

    def get_all_tools(self) -> list[BaseTool]:
        """Return all tools (for Master's planning context)."""
        return self.all_tools
```

---

## Memory System

### Session Memory (Per-Tab, SQLite)

Each session/tab has its own conversation history and context:

```sql
CREATE TABLE sessions (
    id TEXT PRIMARY KEY,
    title TEXT,
    status TEXT DEFAULT 'active',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    summary TEXT,                    -- Auto-generated on completion
    summary_embedding_id TEXT        -- ChromaDB ref for cross-session search
);

CREATE TABLE messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL REFERENCES sessions(id),
    role TEXT NOT NULL,              -- 'user', 'assistant', 'tool', 'system'
    content TEXT NOT NULL,
    agent_name TEXT,                 -- Which specialist agent produced this
    tool_name TEXT,                  -- Which MCP tool was called
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE session_context (
    session_id TEXT NOT NULL,
    key TEXT NOT NULL,               -- 'last_email_id', 'current_document', etc.
    value TEXT NOT NULL,
    PRIMARY KEY (session_id, key)
);

CREATE TABLE artifacts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL REFERENCES sessions(id),
    type TEXT NOT NULL,              -- 'email_draft', 'calendar_event', 'file', 'summary'
    data TEXT NOT NULL,              -- JSON serialized
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### Long-Term Memory (Shared, ChromaDB)

Cross-session knowledge store with these collections:

| Collection | What It Stores | Used By |
|-----------|---------------|---------|
| `session_summaries` | Auto-generated session summaries with embeddings | Master (cross-session retrieval) |
| `entities` | People, companies, topics learned about | Research Agent, Master |
| `documents` | Processed document summaries and chunks | File Handler, Master |
| `preferences` | Learned user preferences (communication style, etc.) | Email Agent, Master |

### Episodic Memory (Cross-Session, SQLite + ChromaDB)

```sql
CREATE TABLE episodes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    task_type TEXT NOT NULL,         -- 'email_summary', 'meeting_prep', 'deadline_extract'
    user_query TEXT NOT NULL,
    plan_json TEXT NOT NULL,         -- The Master's plan that succeeded
    agents_used TEXT NOT NULL,       -- JSON array of agent names
    outcome TEXT NOT NULL,           -- 'success', 'partial', 'failed'
    duration_seconds REAL,
    embedding_id TEXT,               -- ChromaDB for semantic search
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE strategy_patterns (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    pattern_name TEXT NOT NULL,
    trigger_description TEXT NOT NULL,
    plan_template TEXT NOT NULL,     -- Reusable plan JSON
    success_rate REAL DEFAULT 0.0,
    use_count INTEGER DEFAULT 0,
    last_used DATETIME
);
```

### Reference Resolution

Maps anaphoric references within a session:
- "it" / "this" / "that" → most recent entity of matching type
- "that email" / "the email" → `session_context['last_email_id']`
- "the document" / "this file" → `session_context['current_document']`
- "them" / "everyone" → `session_context['current_person_list']`

---

## Linkup Integration (Research Agent)

### Smart Query Pipeline

```
User request mentions "Acme Corp" (unknown entity)
  |
  v
[1. Need Detection] -- Master or Research Agent identifies:
    type: "entity_unknown", entity: "Acme Corp"
  |
  v
[2. Query Formulation] -- Type-specific strategy:
    Query 1: "Acme Corp company overview what does it do" (standard, sourcedAnswer)
    Query 2: "Acme Corp recent news funding 2026" (standard, searchResults)
  |
  v
[3. Privacy Filter] -- Before sending:
    Strip: email addresses, phone numbers, internal project names
    Replace: "john.doe@company.com mentioned Acme Corp" -> "Acme Corp company overview"
  |
  v
[4. Execute] -- Linkup MCP: linkup_search for each query
  |
  v
[5. Synthesize] -- LLM combines results:
    {
      "summary": "Acme Corp is a Series B fintech startup...",
      "key_facts": ["Founded 2021", "Raised $45M Series B", ...],
      "sources": [{"url": "...", "title": "...", "relevance": "high"}],
      "confidence": 0.92
    }
  |
  v
[6. Store] -- Save entity info in ChromaDB entities collection
```

---

## Flet UI Design

### Layout

```
+=============================================================================+
|  ATLAS                                    [Settings] [Theme] [+ New Session]|
+=============================================================================+
| [Session 1: Email Summary] | [Session 2: Meeting Prep] | [Session 3] | [+] |
+=============================================================================+
|                              |                                              |
|    REASONING TRACE PANEL     |              CHAT PANEL                      |
|    (Left, ~30% width)       |              (Right, ~70% width)             |
|                              |                                              |
|  +--------------------------+|  +------------------------------------------+|
|  | [Planning...]            ||  |                                          ||
|  |   Intent: email summary  ||  |  [User] "Summarize my unread emails     ||
|  |   Agents: Email, Calendar||  |   and extract deadlines"                 ||
|  |                          ||  |                                          ||
|  | [Email Agent: reading]   ||  |  [ATLAS] Found 7 unread emails:         ||
|  |   Tool: gmail_search     ||  |   1. Acme Corp - Partnership deadline   ||
|  |   Found: 7 unread        ||  |   2. Prof Kumar - Project due Feb 28    ||
|  |                          ||  |   ...                                    ||
|  | [Research Agent: lookup]  ||  |                                          ||
|  |   Linkup: "Acme Corp"   ||  |  +-- Tool Card: Gmail ----------------+ ||
|  |   Privacy: query sanitized||  |  | 7 emails fetched, 3 deadlines      | ||
|  |                          ||  |  +------------------------------------+ ||
|  | [Calendar Agent: creating]||  |                                          ||
|  |   3 events created       ||  |  +-- Tool Card: Calendar ---------------+||
|  |                          ||  |  | 3 events created successfully        |||
|  | [Evaluation: PASSED]     ||  |  +------------------------------------+ ||
|  +--------------------------+|  |                                          ||
|                              |  +------------------------------------------+|
|  SESSION CONTEXT (collapsed) |                                              |
|  +--------------------------+|  +------------------------------------------+|
|  | Entities: Acme Corp,     ||  | [FilePicker: Upload PDF/DOCX/Image]     ||
|  |   Prof Kumar             ||  +------------------------------------------+|
|  | Past sessions: 2 related ||                                              |
|  +--------------------------+|  +------------------------------------------+|
|                              |  | [Message Input]                  [Send]  ||
|                              |  +------------------------------------------+|
+=============================================================================+
```

### Flet Implementation

```python
import flet as ft

class ATLASApp:
    def __init__(self, page: ft.Page):
        self.page = page
        self.page.title = "ATLAS - Desktop Intelligence Agent"
        self.page.theme_mode = ft.ThemeMode.DARK
        self.sessions: dict[str, SessionTab] = {}

    def build(self):
        # Top: Tab bar for sessions
        self.tabs = ft.Tabs(
            selected_index=0,
            on_change=self.on_tab_change,
            tabs=[]
        )

        # Each tab contains a Row with:
        # Left: ReasoningTracePanel (ListView)
        # Right: Column with ChatPanel (ListView) + InputRow

        self.page.add(
            ft.Row([
                ft.TextButton("+ New Session", on_click=self.new_session)
            ]),
            self.tabs
        )

    def new_session(self, e):
        session = SessionTab(session_id=uuid4(), page=self.page)
        self.tabs.tabs.append(session.tab)
        self.page.update()

class SessionTab:
    """One tab = one session with its own chat + reasoning trace."""

    def __init__(self, session_id, page):
        self.session_id = session_id
        self.reasoning_list = ft.ListView(auto_scroll=True, expand=True)
        self.chat_list = ft.ListView(auto_scroll=True, expand=True)
        self.input_field = ft.TextField(
            hint_text="Ask ATLAS anything...",
            shift_enter=True,
            expand=True,
            on_submit=self.send_message
        )

        self.tab = ft.Tab(
            text=f"Session {session_id[:8]}",
            content=ft.Row([
                # Left: Reasoning trace
                ft.Container(
                    content=self.reasoning_list,
                    width=300,
                    bgcolor=ft.Colors.SURFACE_CONTAINER,
                ),
                ft.VerticalDivider(),
                # Right: Chat + input
                ft.Column([
                    self.chat_list,
                    ft.Row([
                        self.input_field,
                        ft.IconButton(ft.Icons.SEND, on_click=self.send_message),
                        ft.IconButton(ft.Icons.ATTACH_FILE, on_click=self.pick_file),
                    ])
                ], expand=True),
            ], expand=True)
        )

    async def send_message(self, e):
        user_text = self.input_field.value
        self.input_field.value = ""
        self.page.update()

        # Add user message to chat
        self.chat_list.controls.append(UserBubble(user_text))
        self.page.update()

        # Run agent in background (non-blocking)
        self.page.run_task(self.run_agent, user_text)

    async def run_agent(self, user_text: str):
        """Runs Master Orchestrator — updates UI via callbacks."""
        # Master processes, streams reasoning steps back to UI
        async for step in master.stream(user_text, session_id=self.session_id):
            if step.type == "reasoning":
                self.reasoning_list.controls.append(ReasoningStep(step))
            elif step.type == "response":
                self.chat_list.controls.append(AgentBubble(step))
            elif step.type == "tool_card":
                self.chat_list.controls.append(ToolCard(step))
            self.page.update()
```

### Async Pattern

Flet's `page.run_task()` runs coroutines in the event loop without blocking the UI. The Master Orchestrator yields reasoning steps as an async generator, and each step updates the UI immediately.

---

## Project Structure

```
Desktop-agent/
├── atlas/                           # Main package
│   ├── __init__.py
│   ├── app.py                       # Flet app entry point, wiring
│   │
│   ├── master/                      # Master Orchestrator
│   │   ├── __init__.py
│   │   ├── orchestrator.py          # LangGraph StateGraph (CRITICAL)
│   │   ├── state.py                 # MasterState TypedDict
│   │   ├── planner.py               # Planning node (LLM-powered)
│   │   ├── evaluator.py             # Evaluation node (LLM-powered)
│   │   ├── routing.py               # Conditional edge functions
│   │   └── prompts.py               # System prompts for Master
│   │
│   ├── agents/                      # Specialist Agents
│   │   ├── __init__.py
│   │   ├── base.py                  # SpecialistAgent ABC
│   │   ├── email_agent.py           # Gmail integration (hybrid/LLM)
│   │   ├── calendar_agent.py        # Google Calendar (simple)
│   │   ├── file_agent.py            # PDF/DOCX/OCR processing (simple)
│   │   ├── research_agent.py        # Linkup web research (hybrid/LLM)
│   │   ├── translator_agent.py      # Translation (hybrid/LLM)
│   │   ├── whatsapp_agent.py        # WhatsApp messaging (simple)
│   │   ├── macos_agent.py           # macOS automation (simple)
│   │   └── registry.py              # Agent registry + capability catalog
│   │
│   ├── mcp/                         # MCP integration layer
│   │   ├── __init__.py
│   │   ├── manager.py               # MCPManager (CRITICAL)
│   │   ├── config_loader.py         # YAML config with env var resolution
│   │   └── health.py                # Server health monitoring
│   │
│   ├── llm/                         # LLM abstraction
│   │   ├── __init__.py
│   │   ├── provider.py              # OllamaProvider, ClaudeProvider, OpenAIProvider
│   │   └── router.py                # LLMRouter (local/cloud switching)
│   │
│   ├── memory/                      # Three-tier memory
│   │   ├── __init__.py
│   │   ├── manager.py               # MemoryManager (CRITICAL)
│   │   ├── session_store.py         # Per-session SQLite (messages, context, artifacts)
│   │   ├── long_term.py             # ChromaDB (entities, documents, preferences)
│   │   ├── episodic.py              # Episodes + strategy patterns
│   │   ├── cross_session.py         # Cross-session search + retrieval
│   │   ├── reference_resolver.py    # "it"/"that email" resolution
│   │   └── embedder.py              # Local sentence-transformers
│   │
│   ├── research/                    # Linkup integration
│   │   ├── __init__.py
│   │   ├── engine.py                # LinkupResearchEngine (CRITICAL)
│   │   ├── query_formulator.py      # Smart query generation per need type
│   │   ├── synthesizer.py           # Multi-source result synthesis
│   │   └── privacy_filter.py        # PII stripping from queries
│   │
│   ├── documents/                   # Document processing
│   │   ├── __init__.py
│   │   ├── processor.py             # Unified interface
│   │   ├── pdf_handler.py           # PyMuPDF extraction
│   │   ├── docx_handler.py          # python-docx extraction
│   │   └── image_handler.py         # pytesseract OCR
│   │
│   ├── ui/                          # Flet UI
│   │   ├── __init__.py
│   │   ├── main.py                  # ATLASApp, page setup (CRITICAL)
│   │   ├── session_tab.py           # SessionTab (chat + reasoning per tab)
│   │   ├── chat_panel.py            # Message bubbles, tool cards
│   │   ├── reasoning_panel.py       # Reasoning step widgets
│   │   ├── settings_view.py         # LLM/MCP/privacy settings
│   │   ├── components.py            # Reusable widgets (UserBubble, AgentBubble, etc.)
│   │   └── theme.py                 # Colors, fonts, dark/light mode
│   │
│   └── utils/
│       ├── __init__.py
│       ├── config.py                # App-wide YAML config loader
│       ├── logging.py               # Structured logging
│       └── security.py              # Keychain credential storage
│
├── config/
│   ├── mcp_servers.yaml             # MCP server definitions (agent-owned + shared)
│   ├── llm_config.yaml              # LLM provider settings
│   └── app_config.yaml              # General app settings
│
├── data/                            # Local storage (gitignored)
│   ├── atlas.db                     # SQLite (sessions, episodes, strategies)
│   ├── chroma/                      # ChromaDB persistent storage
│   └── logs/                        # Application logs
│
├── tests/
│   ├── test_master/                 # Orchestrator tests
│   ├── test_agents/                 # Each specialist agent
│   ├── test_mcp/                    # MCP manager
│   ├── test_memory/                 # All memory tiers
│   ├── test_research/               # Linkup engine
│   └── fixtures/                    # Sample emails, PDFs, etc.
│
├── scripts/
│   ├── setup_ollama.sh              # Ollama + Qwen3 setup
│   └── setup_google_oauth.sh        # Google OAuth guide
│
├── requirements.txt
├── pyproject.toml
├── .env.example
├── Makefile
└── .gitignore
```

---

## 5 Killer Demo Scenarios

### Demo 1: "Executive Briefing" (Generality + Autonomy + Information Synthesis)
**User:** "Prepare a briefing for my meeting with TechNova Inc tomorrow"
**Master plan:** CalendarAgent.list_events -> EmailAgent.search("TechNova") -> ResearchAgent.research_entity("TechNova") + ResearchAgent.research_person(attendees) -> FileHandlerAgent.write_file(briefing)
**Shows:** 4 agents coordinated, Linkup deep research, cross-domain reasoning

### Demo 2: "Follow-Up Chain" (Context Awareness)
4-message chain in same session tab:
1. "Show me the email from Prof Kumar" -> EmailAgent reads
2. "When is the deadline in it?" -> Reference resolution ("it"), EmailAgent.extract_deadlines
3. "Add that to my calendar and draft a reply" -> CalendarAgent + EmailAgent.draft_reply
4. "Summarize the attachment" -> FileHandlerAgent processes PDF
**Shows:** Reference resolution, multi-turn continuity, 3 agents in one session

### Demo 3: "Fact-Checking Analyst" (Linkup showcase)
**User uploads PDF:** "Verify the key statistics using web research"
**Flow:** FileHandlerAgent.read_pdf -> ResearchAgent.verify_claim (x5 queries, privacy-sanitized) -> "3 verified, 1 outdated, 1 unverifiable" with sources
**Shows:** Privacy filter visible in reasoning trace, multi-query Linkup chaining

### Demo 4: "Cross-Session Knowledge Transfer" (Episodic memory)
**Session 2** (after Demo 2 completed in Session 1):
**User:** "Extract deadlines from this contract, like you did with my emails"
**Flow:** Master finds "deadline_extraction" strategy in episodic memory from Session 1 -> adapts for contract -> proactively offers to calendar them
**Shows:** Cross-session context retrieval, strategy reuse, proactive suggestions

### Demo 5: "Multi-Domain Polyglot Workflow" (Breadth + Privacy)
**User:** "Summarize my unread emails, translate the summary to Spanish, and send it to my study group on WhatsApp"
**Flow:** EmailAgent.read + summarize -> TranslatorAgent.translate(Spanish) -> WhatsApp confirmation dialog -> WhatsAppAgent.send_group_message
**Shows:** 3 agents chained, Translator agent, confirmation before send, local LLM for all processing

---

## 4-Week Sprint Plan

### Team Roles
- **Person A (Lead):** Master Orchestrator, LangGraph, agent coordination
- **Person B:** Memory system, Research Agent (Linkup), document processing
- **Person C:** Flet UI, session management, async bridge
- **Person D:** MCP setup, specialist agents, testing, Google OAuth

### Week 1: Foundation
| | Person A | Person B | Person C | Person D |
|---|---|---|---|---|
| Days 1-2 | Repo setup, project structure, pyproject.toml, Makefile | SQLite schema, ChromaDB init, MemoryManager skeleton | Flet app shell, dark theme, tab bar, basic chat panel | Ollama + Qwen3 setup, Google OAuth credentials |
| Days 3-4 | MCPManager with MultiServerMCPClient, connect Filesystem MCP | Session store: messages, context, artifacts | SessionTab: chat ListView, input field, send button | Gmail MCP + Calendar MCP running standalone |
| Days 5-6 | LLMRouter + OllamaProvider, test Qwen3 tool-calling | Long-term memory: ChromaDB collections, embedder setup | page.run_task async bridge working, non-blocking agent calls | Linkup MCP running, all MCP servers tested individually |
| Day 7 | Basic Master orchestrator: intent -> single agent dispatch -> response | MemoryManager.build_context_for_query working | Chat input -> async agent -> response in chat panel | Integration: MCPManager connects to all servers simultaneously |

**Milestone:** User types in Flet -> Master dispatches to FileHandlerAgent -> Filesystem MCP tool call -> result in chat

### Week 2: Core Agents
| | Person A | Person B | Person C | Person D |
|---|---|---|---|---|
| Days 8-9 | Full Master planning node with LLM, multi-agent dispatch | Episodic memory: store_episode, find_similar, strategy_patterns | Reasoning trace panel: step cards with status icons, live updates | EmailAgent: read, search, summarize (hybrid with LLM) |
| Days 10-11 | Routing logic: research-first, confirmation flow, retry | Reference resolver: "it"/"that email" pattern matching | Tool execution cards in chat panel | CalendarAgent + FileHandlerAgent (simple agents) |
| Days 12-13 | Evaluator node: completeness/correctness checks | LinkupResearchEngine: query formulation, privacy filter | File picker integration (Flet FilePicker) | ResearchAgent: multi-query chaining, synthesis |
| Day 14 | Master -> Agent integration test (Email + Calendar + Research) | Document processing: pdf_handler, docx_handler, image_handler | Session tab title auto-generation | TranslatorAgent + WhatsAppAgent |

**Milestone:** "Summarize my emails" works end-to-end: Master -> EmailAgent -> Gmail MCP -> LLM summarize -> visible reasoning trace

### Week 3: Integration & Intelligence
| | Person A | Person B | Person C | Person D |
|---|---|---|---|---|
| Days 15-16 | Human confirmation flow (Flet dialog before destructive actions) | Cross-session context: auto-index, search past sessions | Polish: message markdown rendering, dark/light toggle | macOS Automation Agent, Demo 1 end-to-end |
| Days 17-18 | Cloud LLM integration (ClaudeProvider), consent flow | Knowledge transfer: episodic strategy reuse in Master planner | Session context panel (collapsible, shows entities + past sessions) | Demo 2 + Demo 3 end-to-end |
| Days 19-20 | Multi-agent parallel dispatch for independent tasks | Preference learning: detect user patterns, inject into prompts | Streaming token display for LLM responses | Demo 4 + Demo 5 end-to-end |
| Day 21 | Error handling: agent failures, MCP reconnection, graceful degradation | Full memory integration test across all tiers | Full UI integration test: all panels updating correctly | All demos working, edge case fixes |

**Milestone:** All 5 demo scenarios work end-to-end with visible reasoning, cross-session context, and privacy features

### Week 4: Polish & Demo Prep
| | Person A | Person B | Person C | Person D |
|---|---|---|---|---|
| Days 22-23 | Performance: parallel agent dispatch, caching, response time optimization | Memory cleanup, embedding cache, query optimization | UI animations, loading states, notification banners | Unit tests: 70%+ coverage on core modules |
| Days 24-25 | Architecture diagram (clean, presentation-ready) | Documentation: memory system, Linkup integration explanation | Setup guide with screenshots | Integration tests with mock MCP servers |
| Days 26-27 | Dry run all 5 demos, fix issues | Dry run support, edge case fixes | Demo recording, presentation materials | README, .env.example, final testing |
| Day 28 | **FINAL: Full demo rehearsal. Tag release.** |||

**Milestone:** Production-quality demos, comprehensive documentation, clean codebase

---

## Dependencies

### Python (requirements.txt)
```
# Core Framework
langgraph>=0.2.0
langchain-core>=0.3.0
langchain-mcp-adapters>=0.1.0
langchain-ollama>=0.2.0
langchain-anthropic>=0.3.0
langchain-openai>=0.2.0

# MCP
mcp>=1.0.0

# Linkup
linkup-sdk>=0.2.0

# LLM
ollama>=0.3.0
sentence-transformers>=3.0.0

# Memory
chromadb>=0.5.0

# Document Processing
pymupdf>=1.24.0
python-docx>=1.1.0
pytesseract>=0.3.10
Pillow>=10.0.0

# UI
flet>=0.25.0

# Utilities
pyyaml>=6.0
python-dotenv>=1.0.0
pydantic>=2.0
aiohttp>=3.9.0
keyring>=25.0.0

# Dev
pytest>=8.0.0
pytest-asyncio>=0.23.0
ruff>=0.5.0
```

### System (macOS)
```bash
brew install tesseract node poppler
curl -fsSL https://ollama.com/install.sh | sh
ollama pull qwen3:8b
pip install flet
```

---

## Risk Mitigation

| Risk | Mitigation |
|---|---|
| Qwen3 tool calling unreliable | Fallback: JSON-in-system-prompt pattern. Alternative: Qwen2.5 7B (proven). |
| Google OAuth complex setup | Detailed setup script. Mock mode for demo that simulates Gmail/Calendar responses. |
| MCP server crashes mid-demo | MCPManager reconnection logic. Pre-warm all servers. Graceful error messages. |
| Flet async issues | Use page.run_task() consistently. All UI updates on main thread via page.update(). |
| Cross-session context too noisy | Relevance threshold on ChromaDB similarity search. Manual override option. |
| 7 agents too many to build | Priority order: Email > Calendar > File > Research > Translator > WhatsApp > macOS. Cut macOS/WhatsApp if behind schedule. |

---

## Verification Plan

1. **Unit Tests**: Each agent, memory tier, MCP manager tested with pytest
2. **Integration Tests**: Master -> Agent -> MCP pipeline with mock servers
3. **Demo Tests**: Run all 5 demos end-to-end before presentation
4. **Privacy Audit**: Verify no PII in Linkup queries using test data
5. **Performance**: Simple tasks < 30s, complex research < 2min
6. **Cross-Session**: Verify Session 2 can find and use context from Session 1
7. **UI**: All panels update correctly, no freezes, dark mode works
