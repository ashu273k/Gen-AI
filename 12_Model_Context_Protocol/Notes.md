# GenAI — Session 12: Model Context Protocol (MCP) & Agent Communication

---

## 1. What is MCP?

Let's start with the problem it solves.

By Session 7, you learned that agents use tools. But consider the reality of building a production agent that needs to connect to:

```
Gmail        → read/send emails
Google Drive → read/write documents  
GitHub       → read/write code
Slack        → send messages
Notion       → read/write pages
PostgreSQL   → query databases
Stripe       → check payments
Jira         → manage tickets
```

### The Problem Before MCP:

```
Every tool integration was CUSTOM:

Agent Team A builds Gmail integration:
  → Custom auth flow
  → Custom message format
  → Custom error handling
  → Hardcoded to their agent framework

Agent Team B also needs Gmail:
  → Builds it again from scratch
  → Different auth flow
  → Different message format
  → Not reusable

100 companies × 50 integrations = 5,000 custom integrations
All incompatible. None reusable. Massive duplication.
```

### What MCP Solves:

```
MCP = A STANDARD PROTOCOL for agents to connect to tools

Like HTTP standardised web communication:
  Before HTTP: every website had its own connection protocol
  After HTTP:  any browser talks to any website

Like USB standardised device connection:
  Before USB: every device needed its own port
  After USB:  any device plugs into any computer

MCP does the same for AI agents and tools:
  Before MCP: every agent-tool connection was custom
  After MCP:  any MCP-compatible agent connects to 
              any MCP-compatible tool server
```

> **MCP = the USB standard for AI agents and tools.**

Anthropic open-sourced MCP in November 2024. It has since been adopted by OpenAI, Google DeepMind, Microsoft, and hundreds of tool providers — making it the de facto industry standard.

---

## 2. MCP Architecture — The Three Roles

MCP defines three distinct roles in every interaction:

```
┌─────────────────────────────────────────────────────────┐
│                   MCP ARCHITECTURE                       │
│                                                          │
│  ┌─────────────────────┐                                │
│  │    MCP HOST          │                               │
│  │  (Claude, GPT,       │                               │
│  │   your custom agent) │                               │
│  │                      │                               │
│  │  - Runs the LLM      │                               │
│  │  - Decides which     │                               │
│  │    tools to call     │                               │
│  │  - Manages context   │                               │
│  └──────────┬───────────┘                               │
│             │  MCP Protocol                             │
│             ↓  (JSON-RPC over stdio/HTTP/SSE)           │
│  ┌─────────────────────┐                                │
│  │    MCP CLIENT        │                               │
│  │  (built into host)   │                               │
│  │                      │                               │
│  │  - Speaks MCP        │                               │
│  │  - Manages server    │                               │
│  │    connections       │                               │
│  │  - Routes tool calls │                               │
│  └──────────┬───────────┘                               │
│             │  Standard MCP messages                    │
│             ↓                                           │
│  ┌─────────────────────┐  ┌─────────────────────┐      │
│  │    MCP SERVER        │  │    MCP SERVER        │     │
│  │   (Gmail)            │  │   (GitHub)           │     │
│  │                      │  │                      │     │
│  │  - Exposes tools     │  │  - Exposes tools     │     │
│  │  - Handles auth      │  │  - Handles auth      │     │
│  │  - Executes actions  │  │  - Executes actions  │     │
│  └─────────────────────┘  └─────────────────────┘      │
└─────────────────────────────────────────────────────────┘
```

### The Three Roles:

**MCP Host**
- The AI application running the LLM (Claude.ai, your custom agent)
- Decides which MCP servers to connect to
- The LLM inside the host reasons about which tools to use

**MCP Client**
- Lives inside the host
- Manages the actual connections to MCP servers
- Translates LLM tool calls into MCP protocol messages
- Usually transparent — you don't build this, it's part of the framework

**MCP Server**
- An independent process exposing tools over the MCP protocol
- Each server is a self-contained integration (Gmail server, GitHub server)
- Can be local (runs on your machine) or remote (runs in the cloud)
- **This is what you build when creating integrations**

---

## 3. What MCP Servers Expose — The Three Primitives

Every MCP server can expose up to three types of capabilities:

### Primitive 1 — Tools
Actions the agent can **execute** — equivalent to function calls.

```json
{
  "name": "send_email",
  "description": "Send an email to one or more recipients",
  "inputSchema": {
    "type": "object",
    "properties": {
      "to": {"type": "array", "items": {"type": "string"}},
      "subject": {"type": "string"},
      "body": {"type": "string"},
      "attachments": {"type": "array", "items": {"type": "string"}}
    },
    "required": ["to", "subject", "body"]
  }
}
```

### Primitive 2 — Resources
Data the agent can **read** — files, database records, API responses.

```json
{
  "uri": "gmail://inbox/unread",
  "name": "Unread Gmail messages",
  "description": "All unread messages in the Gmail inbox",
  "mimeType": "application/json"
}
```

Resources are like **read-only files** the agent can access. The agent reads them as context, not as actions.

### Primitive 3 — Prompts
Pre-built **prompt templates** the server exposes for common tasks.

```json
{
  "name": "summarise_thread",
  "description": "Summarise a Gmail email thread",
  "arguments": [
    {
      "name": "thread_id",
      "description": "The Gmail thread ID to summarise",
      "required": true
    }
  ]
}
```

---

## 4. The MCP Protocol — How Messages Flow

MCP uses **JSON-RPC 2.0** as its message format — a lightweight, well-established protocol for remote procedure calls.

### The Handshake (Connection Establishment):

```
Host → Server:  initialize request
                {capabilities, clientInfo, protocolVersion}

Server → Host:  initialize response  
                {capabilities, serverInfo, protocolVersion}

Host → Server:  initialized notification
                (confirms handshake complete)

Connection established ✅
```

### A Tool Call in MCP:

```
1. LLM decides to call send_email

2. Host → Server: tools/call request
   {
     "jsonrpc": "2.0",
     "id": "req_001",
     "method": "tools/call",
     "params": {
       "name": "send_email",
       "arguments": {
         "to": ["priya@company.com"],
         "subject": "Q3 Report",
         "body": "Please find attached..."
       }
     }
   }

3. Server executes the actual Gmail API call

4. Server → Host: tools/call response
   {
     "jsonrpc": "2.0", 
     "id": "req_001",
     "result": {
       "content": [
         {
           "type": "text",
           "text": "Email sent successfully. Message ID: 18c4f2a9b3e1"
         }
       ]
     }
   }

5. Host feeds result back to LLM context
6. LLM generates response to user ✅
```

### Transport Options:

```
stdio (Standard Input/Output):
  → Host launches server as a subprocess
  → Communication via stdin/stdout pipes
  → Best for: local tools, CLI integrations
  → Example: filesystem server, local database

HTTP + SSE (Server-Sent Events):
  → Server runs as an HTTP service
  → Client connects via HTTP, server streams back via SSE
  → Best for: remote servers, cloud tools
  → Example: Gmail server, GitHub server, Slack server

WebSocket (emerging):
  → Bidirectional real-time communication
  → Best for: high-frequency tool calls, streaming results
```

---

## 5. Context Passing in MCP

One of MCP's key innovations is **rich context passing** — the server doesn't just receive a function call, it receives the full context needed to execute intelligently.

```python
# Traditional tool call (just arguments):
send_email(to="priya@company.com", subject="Meeting", body="...")

# MCP tool call (arguments + context):
{
  "name": "send_email",
  "arguments": {
    "to": "priya@company.com",
    "subject": "Meeting",
    "body": "..."
  },
  "_meta": {
    "user_id": "rahul_123",
    "conversation_id": "conv_abc",
    "timestamp": "2025-06-01T10:30:00",
    "permissions": ["gmail:send", "gmail:read"]
  }
}
```

The server can use this context to:
- Apply user-specific permissions
- Log actions to the right audit trail
- Apply user preferences (e.g. always CC manager on important emails)
- Rate limit per user

---

## 6. Agent-to-Agent (A2A) Communication

This is where MCP goes beyond simple tool calling into **multi-agent coordination**.

### The Problem:

```
You have three specialised agents:
  Research Agent    → finds information
  Analysis Agent    → processes data
  Writing Agent     → creates documents

How do they communicate?
How does Research Agent hand off to Analysis Agent?
How does Analysis Agent send results to Writing Agent?
```

### A2A with MCP:

Each agent **exposes itself as an MCP server**. Other agents connect to it as an MCP client. Agents become both clients (calling other agents) and servers (being called by other agents).

```
┌─────────────────────────────────────────────────────────┐
│              AGENT-TO-AGENT ARCHITECTURE                 │
│                                                          │
│  ┌─────────────────┐                                    │
│  │  ORCHESTRATOR   │                                    │
│  │     AGENT       │                                    │
│  │  (MCP Client)   │                                    │
│  └────────┬────────┘                                    │
│           │ MCP Protocol                                │
│    ┌──────┴───────┐                                     │
│    ↓              ↓                                     │
│  ┌──────────┐  ┌──────────┐                            │
│  │ Research │  │ Analysis │                            │
│  │  Agent   │  │  Agent   │                            │
│  │(MCP Srv) │  │(MCP Srv) │                            │
│  └────┬─────┘  └────┬─────┘                            │
│       │              │                                  │
│       ↓              ↓                                  │
│  ┌──────────┐  ┌──────────┐                            │
│  │ Web      │  │ Database │                            │
│  │ Search   │  │  Server  │                            │
│  │(MCP Srv) │  │(MCP Srv) │                            │
│  └──────────┘  └──────────┘                            │
└─────────────────────────────────────────────────────────┘
```

### A2A Message Flow:

```
Orchestrator: "Research the top 3 AI companies and 
               analyse their financials"

Step 1: Orchestrator → Research Agent (MCP call)
  {
    "name": "research_topic",
    "arguments": {
      "query": "top 3 AI companies by revenue 2025",
      "depth": "comprehensive",
      "output_format": "structured_json"
    }
  }

Step 2: Research Agent executes:
  → Calls Web Search MCP server
  → Calls Wikipedia MCP server
  → Compiles findings
  → Returns structured JSON to Orchestrator

Step 3: Orchestrator → Analysis Agent (MCP call)
  {
    "name": "analyse_financials",
    "arguments": {
      "companies": [research_results],
      "metrics": ["revenue", "growth", "market_cap"],
      "comparison_type": "side_by_side"
    }
  }

Step 4: Analysis Agent executes:
  → Calls Database MCP server for financial data
  → Runs analysis
  → Returns analysis to Orchestrator

Step 5: Orchestrator composes final response ✅
```

---

## 7. Orchestrator-Subagent Patterns

MCP enables several proven patterns for multi-agent systems:

### Pattern 1 — Hub and Spoke (Star Topology)

```
         [Orchestrator]
        /      |       \
[Agent A] [Agent B] [Agent C]

Orchestrator coordinates all agents directly.
Agents don't know about each other.
Best for: clear task decomposition, independent subtasks
```

```python
orchestrator = Agent(
    name="Orchestrator",
    instructions="""
    You coordinate a team of specialist agents.
    For research tasks: delegate to Research Agent
    For data tasks: delegate to Analysis Agent  
    For writing tasks: delegate to Writing Agent
    Combine their outputs into a coherent final response.
    """,
    handoffs=[
        handoff(research_agent),
        handoff(analysis_agent),
        handoff(writing_agent)
    ]
)
```

### Pattern 2 — Pipeline (Sequential)

```
[Input] → [Agent A] → [Agent B] → [Agent C] → [Output]

Each agent processes and passes to the next.
Output of one is input to the next.
Best for: transformation pipelines, document processing
```

```python
# Each agent's output becomes next agent's input
async def pipeline(user_input: str) -> str:
    
    # Stage 1: Research
    research = await Runner.run_async(
        research_agent, user_input
    )
    
    # Stage 2: Analysis (takes research output as input)
    analysis = await Runner.run_async(
        analysis_agent, research.final_output
    )
    
    # Stage 3: Writing (takes analysis output as input)
    final = await Runner.run_async(
        writing_agent, analysis.final_output
    )
    
    return final.final_output
```

### Pattern 3 — Parallel Fan-Out

```
              [Orchestrator]
             /       |       \
          [A]       [B]       [C]
           \         |        /
              [Orchestrator]
              (merges results)

All agents run simultaneously.
Orchestrator waits for all, then merges.
Best for: independent research tasks, speed-critical workflows
```

```python
import asyncio

async def parallel_research(topic: str) -> str:
    
    # Launch all agents simultaneously
    tasks = [
        Runner.run_async(web_research_agent, topic),
        Runner.run_async(arxiv_agent, topic),
        Runner.run_async(news_agent, topic)
    ]
    
    # Wait for all to complete
    results = await asyncio.gather(*tasks)
    
    # Merge all results
    merged_context = "\n\n".join([r.final_output for r in results])
    
    # Final synthesis
    synthesis = await Runner.run_async(
        synthesis_agent, merged_context
    )
    
    return synthesis.final_output
```

### Pattern 4 — Hierarchical (Tree)

```
              [CEO Agent]
             /            \
    [Research Mgr]    [Writing Mgr]
      /      \             /      \
[Web]  [DB]  [Draft]  [Review]

Multi-level delegation.
Best for: complex enterprise workflows
```

---

## 8. Handoff Protocols

When one agent hands off to another, the **handoff message** must carry enough context for the receiving agent to continue seamlessly.

### Minimal Handoff (Bad):

```python
# Bad — no context
handoff_message = "Research Infosys"
# Receiving agent starts completely fresh ❌
```

### Rich Handoff (Good):

```python
# Good — full context transfer
handoff_message = {
    "task": "Analyse Infosys financials",
    "context": {
        "research_completed": research_results,
        "user_goal": "Investment decision for ₹5 lakh",
        "constraints": "Focus on last 3 quarters",
        "previous_agents": ["Research Agent"],
        "work_done_so_far": "Gathered stock price, news, annual report",
        "remaining_work": "Financial ratio analysis, peer comparison"
    },
    "output_format": "structured JSON with recommendation",
    "deadline": "within 10 tool calls"
}
```

### Standard Handoff Protocol in Agent SDK:

```python
from agents import handoff, RunContextWrapper

def on_handoff_to_analyst(ctx: RunContextWrapper):
    """Called when orchestrator hands off to analyst agent."""
    # Package current context for analyst
    return {
        "research_data": ctx.state.get("research_results"),
        "user_query": ctx.state.get("original_query"),
        "analysis_focus": ctx.state.get("analysis_parameters")
    }

orchestrator = Agent(
    name="Orchestrator",
    handoffs=[
        handoff(
            analyst_agent,
            on_handoff=on_handoff_to_analyst,  # context packager
            tool_name_override="delegate_to_analyst"
        )
    ]
)
```

---

## 9. Building a Production MCP Tool Server

This is the practical skill — building an MCP server that exposes your own tools to any MCP-compatible agent.

```python
from mcp.server import Server
from mcp.server.models import InitializationOptions
from mcp.types import Tool, TextContent
import mcp.server.stdio

# Create the MCP server
server = Server("company-data-server")

# Define available tools
@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="get_employee_info",
            description="""Retrieve information about a company employee.
                          Use when asked about team members, roles, or contacts.""",
            inputSchema={
                "type": "object",
                "properties": {
                    "employee_id": {
                        "type": "string",
                        "description": "Employee ID or email address"
                    },
                    "fields": {
                        "type": "array",
                        "items": {"type": "string"},
                        "description": "Fields to retrieve: name, role, team, manager"
                    }
                },
                "required": ["employee_id"]
            }
        ),
        Tool(
            name="search_company_docs",
            description="""Search internal company documentation.
                          Use for policy questions, process guides, onboarding docs.""",
            inputSchema={
                "type": "object", 
                "properties": {
                    "query": {"type": "string"},
                    "doc_type": {
                        "type": "string",
                        "enum": ["policy", "process", "technical", "all"]
                    }
                },
                "required": ["query"]
            }
        )
    ]

# Implement tool execution
@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    
    if name == "get_employee_info":
        employee_id = arguments["employee_id"]
        fields = arguments.get("fields", ["name", "role", "team"])
        
        # Query your actual employee database
        employee = await employee_db.get(employee_id, fields)
        
        return [TextContent(
            type="text",
            text=str(employee)
        )]
    
    elif name == "search_company_docs":
        query = arguments["query"]
        doc_type = arguments.get("doc_type", "all")
        
        # Query your actual document search system
        results = await doc_search.search(query, doc_type)
        
        return [TextContent(
            type="text",
            text=str(results)
        )]
    
    else:
        raise ValueError(f"Unknown tool: {name}")

# Run the server
async def main():
    async with mcp.server.stdio.stdio_server() as (read, write):
        await server.run(
            read, write,
            InitializationOptions(
                server_name="company-data-server",
                server_version="1.0.0"
            )
        )

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

### Connecting Your Server to Claude:

```json
// claude_desktop_config.json
{
  "mcpServers": {
    "company-data": {
      "command": "python",
      "args": ["/path/to/your/mcp_server.py"],
      "env": {
        "DATABASE_URL": "postgresql://...",
        "API_KEY": "your_api_key"
      }
    }
  }
}
```

Once configured, Claude automatically discovers and uses your tools — no custom integration code needed.

---

## 10. Modern AI Infrastructure Patterns

MCP sits within a broader emerging infrastructure stack for production AI:

```
┌─────────────────────────────────────────────────────────┐
│           MODERN AI INFRASTRUCTURE STACK                 │
│                                                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │  USER INTERFACE LAYER                            │   │
│  │  Web app / Mobile / Voice / Claude.ai            │   │
│  └─────────────────────────────────────────────────┘   │
│                         ↕                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  AGENT LAYER                                     │   │
│  │  Agent SDK / LangGraph / Custom                  │   │
│  └─────────────────────────────────────────────────┘   │
│                         ↕ MCP Protocol                  │
│  ┌─────────────────────────────────────────────────┐   │
│  │  TOOL SERVER LAYER                               │   │
│  │  Gmail MCP / GitHub MCP / DB MCP / Custom MCP   │   │
│  └─────────────────────────────────────────────────┘   │
│                         ↕                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  DATA LAYER                                      │   │
│  │  Vector DB / Graph DB / Relational DB / Cache    │   │
│  └─────────────────────────────────────────────────┘   │
│                         ↕                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  OBSERVABILITY LAYER                             │   │
│  │  Traces / Logs / Metrics / LLM Evaluation       │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## The Complete GenAI Journey — Sessions 1 to 12

```
Session 1:  LLM internals — how transformers predict tokens
Session 2:  Prompt Engineering — steering predictions
Session 3:  AI Agents — LLM + tools + reasoning loop
Session 4:  RAG 1 — giving agents a knowledge base
Session 5:  RAG 2 — making retrieval production-grade
Session 6:  LangChain & LangGraph — orchestration frameworks
Session 7:  Agent SDK — cleaner multi-agent development
Session 8:  Threads & Tracing — memory and observability
Session 9:  Memory Systems — persistence across sessions
Session 10: Graph Databases — relationship-aware knowledge
Session 11: Voice Agents — real-time audio I/O
Session 12: MCP — the standard protocol that connects it all
                   ↑
            This is the infrastructure layer
            that makes all the above interoperable
            at production scale
```

---

## Quick Recap — Session 12

| Concept | One-liner |
|---|---|
| MCP | Universal standard protocol connecting agents to tools — the USB of AI |
| MCP Host | The AI application running the LLM |
| MCP Client | Lives in the host, manages server connections |
| MCP Server | Exposes tools over MCP protocol — what you build |
| Tools | Actions the agent executes via MCP |
| Resources | Read-only data the agent accesses via MCP |
| Prompts | Pre-built templates exposed by MCP servers |
| JSON-RPC | The message format MCP uses |
| Transport | stdio (local) or HTTP+SSE (remote) |
| A2A communication | Agents expose themselves as MCP servers, enabling agent-to-agent calls |
| Hub and spoke | Orchestrator coordinates all agents directly |
| Pipeline | Sequential — output of one agent is input to next |
| Parallel fan-out | All agents run simultaneously, results merged |
| Handoff protocol | Rich context transfer between agents — not just task, but full state |
| Building MCP server | Define tools → implement call handlers → run over stdio or HTTP |

---

Only **Session 13** left — **Deployment, Scaling & Final Demos** — where you take everything built across all 12 sessions and learn how to ship it to production on AWS with proper scaling, caching, CI/CD, and rate limiting. Ready?