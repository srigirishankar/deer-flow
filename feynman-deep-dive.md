# DeerFlow: Feynman-Style Deep Dive & NanoFlow Simplification Plan

## LAYER 0: The One-Sentence Explanation

**DeerFlow is a program that lets an AI assistant run code on a computer, remember things about you, and ask other AI assistants for help — all while keeping everything safely contained.**

If you can understand that sentence, you understand what it does. Everything below is *how*.

---

## LAYER 1: The Five Building Blocks

Imagine you hired a brilliant intern. They need five things:

```
┌─────────────────────────────────────────────────┐
│                  YOU (Frontend)                   │
│              "Hey, analyze this CSV"              │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│              THE BRAIN (Lead Agent)               │
│  Reads your request, decides what to do          │
│  "I need to write Python code to analyze this"   │
└───┬─────────┬──────────┬──────────┬─────────────┘
    │         │          │          │
    ▼         ▼          ▼          ▼
┌───────┐ ┌───────┐ ┌────────┐ ┌────────┐
│SANDBOX│ │MEMORY │ │ TOOLS  │ │HELPERS │
│Run    │ │Remember│ │Search  │ │Sub-    │
│code   │ │things  │ │the web │ │agents  │
│safely │ │about   │ │read    │ │do work │
│       │ │you     │ │files   │ │in      │
│       │ │        │ │etc.    │ │parallel│
└───────┘ └───────┘ └────────┘ └────────┘
```

That's it. Five building blocks. Everything else is plumbing.

---

## LAYER 2: How Each Block Actually Works

### 2.1 The Brain (Lead Agent)

The brain is built with ONE function call:

```python
# This is literally the entire agent creation (agent.py:325-331)
create_agent(
    model=create_chat_model(name="gpt-4o"),     # Which AI model to use
    tools=[bash, read_file, web_search, ...],    # What it can do
    middleware=[...11 middlewares...],            # Rules it follows
    system_prompt="You are DeerFlow...",         # Its personality
    state_schema=ThreadState,                    # What it remembers per conversation
)
```

`create_agent` comes from LangChain. It creates a LangGraph `StateGraph` — a loop:

```
User message → AI thinks → AI calls tool → Tool returns result → AI thinks again → ... → AI responds
```

That loop IS the agent. It's a while-loop with an LLM deciding when to stop.

**The State** is what persists between loops. It's simple:

```python
class ThreadState(AgentState):        # AgentState has: messages (the conversation)
    sandbox: SandboxState | None       # Which sandbox is assigned
    thread_data: ThreadDataState       # File paths for this conversation
    title: str | None                  # Auto-generated conversation title
    artifacts: list[str]               # Files the agent created
    todos: list | None                 # Task list (plan mode)
    uploaded_files: list[dict] | None  # Files the user uploaded
    viewed_images: dict                # Images the agent has seen
```

**Key insight**: The state is just a typed dictionary that flows through the graph. LangGraph checkpoints it after every step. That's how conversations survive server restarts.

### 2.2 The Middleware Chain (The Rules)

Before and after every AI "think" step, 11 middlewares run in order. Think of them like a conveyor belt:

```
USER MESSAGE arrives
    │
    ▼ ThreadDataMiddleware      → Sets up /workspace, /uploads, /outputs directories
    ▼ UploadsMiddleware         → Tells AI what files user uploaded
    ▼ SandboxMiddleware         → Acquires a sandbox (Docker container or local)
    ▼ DanglingToolCallMiddleware → Patches broken tool call history
    ▼ SummarizationMiddleware   → Compresses old messages if conversation is too long
    ▼ TodoListMiddleware        → Adds task-tracking tool (if plan mode)
    ▼ TitleMiddleware           → Auto-generates conversation title
    ▼ MemoryMiddleware          → Queues conversation for memory extraction
    ▼ ViewImageMiddleware       → Injects image data for vision models
    ▼ SubagentLimitMiddleware   → Caps parallel sub-agents at 3
    ▼ ClarificationMiddleware   → Intercepts "need more info" and pauses
    │
    ▼
AI THINKS AND RESPONDS
```

Each middleware has two hooks: `before_model` (pre-process) and `after_model` (post-process).

**Key insight**: Middlewares are the *policy layer*. The agent is dumb by itself — middlewares make it smart about context, memory, safety, and resource limits.

### 2.3 The Sandbox (Safe Code Execution)

The sandbox is an abstraction over "a place where code runs":

```python
class Sandbox(ABC):                    # Abstract interface
    def execute_command(cmd) -> str     # Run bash commands
    def read_file(path) -> str          # Read files
    def write_file(path, content)       # Write files
    def list_dir(path) -> list[str]     # List directories
```

Two implementations:
- **LocalSandbox**: Just runs on the host filesystem (development)
- **AioSandbox**: Runs inside a Docker container (production)

The clever part is **virtual path translation**:

```
Agent thinks it's writing to:     /mnt/user-data/workspace/analysis.py
Actually writes to:               backend/.deer-flow/threads/abc123/user-data/workspace/analysis.py
```

The agent always sees `/mnt/user-data/...`. The sandbox translates to real paths. This means:
- Each conversation gets its own isolated filesystem
- The agent can't accidentally access other users' files
- The same agent code works with local or Docker sandboxes

### 2.4 Memory (Remembering Things About You)

The memory system has three parts:

**Part 1: The Queue** — After each conversation turn, `MemoryMiddleware` captures messages and puts them in a debounced queue (waits 30 seconds, batches updates).

**Part 2: The Extractor** — A separate LLM call analyzes the conversation and extracts:
```json
{
  "user": {
    "workContext": {"summary": "Senior engineer at Acme Corp, works on payments"},
    "personalContext": {"summary": "Prefers Python, dislikes verbose code"},
    "topOfMind": {"summary": "Migrating from REST to GraphQL"}
  },
  "facts": [
    {"content": "Uses PostgreSQL in production", "confidence": 0.9, "category": "knowledge"},
    {"content": "Prefers dark mode", "confidence": 0.7, "category": "preference"}
  ]
}
```

**Part 3: The Injector** — On the next conversation, top facts get injected into the system prompt inside `<memory>` tags. The AI reads these before responding.

**Key insight**: Memory is just "run an LLM on the conversation, extract JSON, save to file, inject into next prompt." That's all it is.

### 2.5 Sub-Agents (Getting Help)

When a task is complex, the lead agent can spawn sub-agents:

```python
# The lead agent calls the task() tool
task(
    description="Research AWS pricing",
    prompt="Find current pricing for EC2, S3, and RDS...",
    subagent_type="general-purpose"
)
```

Behind the scenes:
1. `SubagentExecutor` creates a new agent (same `create_agent` call, fewer middlewares)
2. Submits it to a thread pool (max 3 concurrent)
3. The parent agent blocks until results come back
4. Results are returned as the tool response

Sub-agents inherit the parent's sandbox and thread data, so they can read/write the same files.

**Key insight**: Sub-agents are just "the same agent, running in a background thread, with a focused task." Not a different system — same `create_agent`, same tools, same sandbox.

### 2.6 Tools (What the Agent Can Do)

Tools assemble from four sources:

```python
def get_available_tools():
    return (
        config_tools           # From config.yaml: web_search, web_fetch, etc.
        + builtin_tools        # present_file, ask_clarification, view_image
        + mcp_tools            # From MCP servers (dynamic, external)
        + subagent_tools       # task() — only if subagent_enabled
    )
```

Each tool is just a Python function with a docstring:

```python
@tool("bash")
def bash_tool(runtime, description, command) -> str:
    sandbox = ensure_sandbox_initialized(runtime)
    return sandbox.execute_command(command)
```

The docstring becomes the tool's description for the LLM. The LLM reads it, decides when to use it, and generates the arguments. LangChain handles the JSON serialization.

### 2.7 Skills (Specialized Workflows)

Skills are NOT code. They're markdown files with instructions:

```markdown
---
name: deep-research
description: Multi-angle research methodology
allowed-tools: [bash, web_search, read_file, write_file, present_files]
---

# Deep Research Skill

## When to Use
When the user asks for comprehensive research on a topic...

## Workflow
1. First, identify 3-5 research angles...
2. For each angle, use web_search to find sources...
3. Synthesize findings into a report...
```

The agent reads this file and follows the instructions. That's it — skills are prompt engineering stored in files, not executable code.

---

## LAYER 3: The Full Data Flow

Let's trace a complete request: "Analyze the CSV I uploaded"

```
1. FRONTEND sends message via LangGraph SDK → SSE stream
                    │
2. LANGGRAPH SERVER receives it, loads make_lead_agent()
                    │
3. MIDDLEWARE CHAIN (before_model):
   ThreadData   → Creates /workspace, /uploads, /outputs for thread abc123
   Uploads      → Finds sales.csv in uploads, adds to context: "Uploaded: sales.csv"
   Sandbox      → Acquires local sandbox, maps /mnt/user-data → real paths
   Memory       → Injects: "User prefers Python, works with sales data"
                    │
4. LLM THINKS:
   System: "You are DeerFlow... User memory: prefers Python..."
   Human:  "Analyze the CSV I uploaded"
   Context: "Uploaded files: /mnt/user-data/uploads/sales.csv"

   AI decides: I should read the file, then write Python to analyze it.
                    │
5. TOOL CALL #1: read_file("/mnt/user-data/uploads/sales.csv")
   → SandboxMiddleware translates path → reads real file → returns content
                    │
6. LLM THINKS again: "OK, it has columns: date, revenue, region. Let me analyze."
                    │
7. TOOL CALL #2: write_file("/mnt/user-data/workspace/analyze.py", "import pandas...")
   → Writes Python script to workspace
                    │
8. TOOL CALL #3: bash("cd /mnt/user-data/workspace && python analyze.py")
   → Runs the script, produces output and chart.png
                    │
9. TOOL CALL #4: present_files(["/mnt/user-data/outputs/chart.png"])
   → Makes the chart visible to the user
                    │
10. LLM RESPONDS: "Here's your analysis. Revenue grew 15% YoY..."
                    │
11. MIDDLEWARE CHAIN (after_model):
    Title    → Generates title: "Sales CSV Analysis"
    Memory   → Queues: "User works with sales data, analyzed CSV"
    Clarify  → No clarification needed, pass through
                    │
12. SSE STREAM → Frontend renders response + chart
```

---

## LAYER 4: The Infrastructure Layer

```
┌─────────────────────────────────────────────┐
│              NGINX (port 2026)               │
│  /api/langgraph/* → LangGraph (2024)        │
│  /api/*           → Gateway API (8001)      │
│  /*               → Frontend (3000)         │
└──────────┬───────────────┬──────────────────┘
           │               │
    ┌──────┴──────┐ ┌──────┴──────┐
    │  LangGraph  │ │  Gateway    │
    │  Server     │ │  API        │
    │             │ │             │
    │ • Agent     │ │ • Models    │
    │   runtime   │ │ • Skills    │
    │ • Thread    │ │ • Memory    │
    │   state     │ │ • Uploads   │
    │ • SSE       │ │ • MCP       │
    │   streaming │ │ • Artifacts │
    └──────┬──────┘ └─────────────┘
           │
    ┌──────┴──────┐
    │  Checkpoint  │
    │  (SQLite/    │
    │   Postgres)  │
    └─────────────┘
```

Two backend services:
- **LangGraph Server**: Runs the agent, manages conversation threads, streams responses
- **Gateway API**: Everything else — config management, file uploads, skills, memory

They share the same `src/` codebase but run as separate processes.

---

## LAYER 5: What's Actually Complex vs. What's Ceremony

Now the Feynman test: **what could you remove and still have the same thing?**

### Actually Essential (Core Logic)

| Component | Lines of Code | Why It Exists |
|---|---|---|
| `create_agent()` call | ~10 lines | The entire agent IS this call |
| `ThreadState` | ~20 lines | State schema for conversations |
| Sandbox tools (bash, read, write) | ~150 lines | Code execution capability |
| Virtual path translation | ~60 lines | File isolation per conversation |
| System prompt template | ~50 lines | Agent personality and instructions |
| Tool assembly | ~40 lines | Combining tools from different sources |
| Model factory | ~50 lines | Creating LLM instances from config |

**Total essential core: ~380 lines of Python.**

### Important but Removable (Good Features)

| Component | Lines | Can be deferred? |
|---|---|---|
| Memory system | ~350 lines | Yes — nice-to-have, not core |
| Sub-agent executor | ~450 lines | Yes — single agent works fine |
| Middleware chain (11 middlewares) | ~800 lines | Most are optional; 3-4 are essential |
| Skills system | ~200 lines | Yes — just put instructions in prompt |
| MCP integration | ~300 lines | Yes — add tools directly instead |

### Pure Ceremony (Removable Without Loss)

| Component | Lines | Why it exists | Why remove it |
|---|---|---|---|
| ClarificationMiddleware | ~80 | Interrupts on unclear requests | LLM can just ask in text |
| TitleMiddleware | ~60 | Auto-titles conversations | Frontend can do this |
| TodoListMiddleware | ~200 | Plan mode task tracking | Nice-to-have, not core |
| ViewImageMiddleware | ~100 | Injects base64 images | Multimodal models handle this natively |
| DanglingToolCallMiddleware | ~50 | Patches broken history | Defensive fix for edge case |
| IM Channels (Slack, Telegram, Feishu) | ~1500 | Chat platform bridges | Separate concern entirely |
| Gateway API | ~500 | REST API for config | Can be merged with LangGraph server |

---

## LAYER 6: The NanoFlow Blueprint

Like OpenManus → NanoManus, here's the minimal "NanoFlow" that has 90% of the capability in 10% of the code:

### NanoFlow Architecture

```
┌──────────────────────────────────────┐
│            NanoFlow                    │
│                                        │
│  ONE file: agent.py (~300 lines)      │
│                                        │
│  create_agent(                        │
│    model = ChatOpenAI("gpt-4o")      │
│    tools = [                          │
│      bash,                            │
│      read_file,                       │
│      write_file,                      │
│      web_search,                      │
│      present_file                     │
│    ]                                  │
│    middleware = [                      │
│      SandboxMiddleware,   ← Essential │
│      SummarizationMiddleware ← Nice   │
│    ]                                  │
│    system_prompt = PROMPT             │
│    state_schema = ThreadState         │
│  )                                    │
│                                        │
│  ONE file: sandbox.py (~100 lines)    │
│  LocalSandbox + path translation      │
│                                        │
│  ONE file: tools.py (~120 lines)      │
│  bash, read, write, ls, str_replace   │
│                                        │
│  ONE file: config.py (~50 lines)      │
│  Model + tool configuration           │
│                                        │
│  langgraph.json (standard)            │
│                                        │
│  Frontend: Next.js + LangGraph SDK    │
│  (reuse from LangGraph templates)     │
└──────────────────────────────────────┘
```

### NanoFlow: What's In, What's Out

| Feature | In NanoFlow? | Why |
|---|---|---|
| Agent + tool loop | YES | This IS the product |
| Sandbox (local) | YES | Code execution is core |
| Path isolation | YES | Security baseline |
| Web search (Tavily) | YES | Essential capability |
| File read/write/bash | YES | Essential capability |
| present_file | YES | User needs to see outputs |
| Summarization | YES | Prevents context overflow |
| Memory | PHASE 2 | Add after core works |
| Sub-agents | PHASE 2 | Single agent handles 80% of tasks |
| MCP | PHASE 2 | Direct tool integration first |
| Skills | PHASE 3 | Just use prompt engineering initially |
| IM channels | PHASE 3 | Separate concern |
| Docker sandbox | PHASE 3 | Local sandbox for MVP |
| Vision support | PHASE 2 | Modern models handle this |

### NanoFlow Implementation (~600 lines total)

```
nanoflow/
├── backend/
│   ├── langgraph.json          # 10 lines - registers agent
│   ├── agent.py                # 150 lines - agent factory + state + prompt
│   ├── sandbox.py              # 100 lines - local sandbox + path translation
│   ├── tools.py                # 150 lines - bash, read, write, ls, web_search
│   ├── models.py               # 50 lines - model factory from config
│   ├── config.py               # 50 lines - YAML config loader
│   └── config.yaml             # 30 lines - model + tool config
├── frontend/                   # Use LangGraph chatbot template
│   └── (Next.js, ~standard)
└── docker-compose.yaml         # LangGraph server + frontend
```

### What Makes NanoFlow Enterprise-Grade

The simplification doesn't sacrifice quality — it removes ceremony:

1. **LangGraph handles all the hard parts**: Checkpointing, streaming, thread management, tool execution loop. You don't reimpliment these.

2. **E2B replaces the entire sandbox system**: Instead of 500+ lines of sandbox code with Docker/K8s orchestration:
   ```python
   from e2b_code_interpreter import Sandbox
   sandbox = Sandbox()  # SOC2 compliant, US-hosted, done.
   ```

3. **LangGraph Cloud replaces the entire deployment stack**: No Nginx, no Gateway API, no Docker Compose orchestration. One `langgraph deploy` command.

4. **LangSmith replaces custom observability**: Every LLM call, tool call, and agent step is automatically traced.

5. **PostgreSQL checkpointer replaces file-based everything**: State, memory, config — all in one managed database.

### Progressive Enhancement Path

```
Week 1-2: NanoFlow MVP
  └── Agent + sandbox + tools + web search
      └── ~600 lines of Python
      └── Deployable on LangGraph Cloud

Week 3-4: + Memory
  └── PostgreSQL-backed memory (not JSON files)
  └── Same LLM extraction approach, ~200 more lines

Week 5-6: + Sub-agents
  └── Same pattern as DeerFlow but simpler
  └── No need for dual thread pools — use LangGraph's built-in task scheduling
  └── ~150 more lines

Week 7-8: + MCP + Enterprise Auth
  └── langchain-mcp-adapters (already exists)
  └── Auth0/Clerk integration
  └── ~200 more lines

Week 9-10: + Skills + E2B Sandbox
  └── Same SKILL.md format
  └── E2B for production sandboxing
  └── ~200 more lines

Total: ~1,350 lines of Python vs DeerFlow's ~5,000+ lines
       Same capabilities, enterprise-grade, US supply chain
```

---

## LAYER 7: The Deepest Insight

Here's what Richard Feynman would say about DeerFlow:

**"It's a while-loop with an LLM making decisions."**

Everything — the middlewares, the sub-agents, the memory, the skills — is decoration around this core loop:

```
while not done:
    response = llm.think(messages + tools)
    if response.has_tool_call:
        result = execute_tool(response.tool_call)
        messages.append(result)
    else:
        done = True
        return response.text
```

LangGraph implements this loop as a `StateGraph`. The "nodes" are the LLM call and tool execution. The "edges" are the conditional routing (tool call → loop back, no tool call → end).

Everything else is:
- **What goes INTO the loop**: system prompt, memory, skill instructions, uploaded files
- **What happens AROUND the loop**: middleware pre/post processing
- **What the loop can DO**: tools (sandbox, web search, file I/O)
- **How the loop SCALES**: sub-agents (more loops running in parallel)

Once you see it as "a while-loop with plugins," the entire system becomes obvious. And building a simpler version becomes trivial — because LangGraph already gives you the loop, the state management, and the checkpointing. You just add the plugins you need.

---

## Summary Decision Matrix

| Approach | Time | Lines of Code | Enterprise Ready | Risk |
|---|---|---|---|---|
| **Fork DeerFlow** | 4 weeks | 10,000+ | NO (China origin) | High |
| **Full reimplementation** | 22 weeks | 5,000+ | Yes | Medium |
| **NanoFlow (recommended)** | 10 weeks | ~1,350 | Yes | Low |

**Recommendation**: Build NanoFlow. Start with 600 lines. Ship in 2 weeks. Progressively enhance. The LangChain/LangGraph ecosystem does 80% of the heavy lifting — you're just wiring it together with enterprise-grade choices (E2B, Auth0, PostgreSQL, LangSmith).

The enterprises don't care that DeerFlow has 11 middlewares. They care that their data is safe, the system is reliable, and it's not built in China. NanoFlow gives them that with dramatically less complexity.
