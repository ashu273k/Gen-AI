# GenAI — Session 7: Agent SDK — Fundamentals

---

## 1. What Are the Flaws in LangGraph?

Before understanding why Agent SDK exists, you need to understand what LangGraph gets **wrong** in production.

LangGraph is powerful but has real pain points:

### Flaw 1 — Too Much Boilerplate
```python
# LangGraph — you write ALL of this just to define a simple agent

class AgentState(TypedDict):
    messages: List[BaseMessage]
    
def should_continue(state):
    if state["messages"][-1].tool_calls:
        return "tools"
    return "end"

workflow = StateGraph(AgentState)
workflow.add_node("agent", call_model)
workflow.add_node("tools", call_tools)
workflow.set_entry_point("agent")
workflow.add_conditional_edges("agent", should_continue,
    {"tools": "tools", "end": END})
workflow.add_edge("tools", "agent")
app = workflow.compile()

# Just to make an agent that uses tools.
# This is 20+ lines of plumbing for a basic use case.
```

### Flaw 2 — Graph Mental Model Doesn't Match Agent Thinking
```
LangGraph forces you to think in:
  nodes → edges → conditional edges → state transitions

But agents are naturally described as:
  "An assistant that has these tools and 
   hands off to this specialist when needed"

The graph abstraction leaks into your business logic.
```

### Flaw 3 — Multi-Agent Coordination Is Painful
```
In LangGraph, connecting multiple agents requires:
  → Defining inter-agent state schemas
  → Managing message passing manually
  → Custom handoff logic between graphs
  → No standard "agent calls another agent" pattern

It's possible but deeply unergonomic.
```

### Flaw 4 — No Built-in Tracing or Observability
```
When your LangGraph agent fails at step 7:
  → No standard way to see what happened
  → No built-in execution trace
  → Debugging = adding print statements everywhere
```

### Flaw 5 — Steep Learning Curve
```
LangGraph concepts to learn before writing one line of logic:
  StateGraph, TypedDict, add_node, add_edge, 
  add_conditional_edges, compile, checkpointer,
  RunnableConfig, ChannelWrite, ChannelRead...
  
Most developers spend days on scaffolding before
shipping anything useful.
```

> Agent SDK was designed to fix all five of these. Same power, dramatically better developer experience.

---

## 2. Agent SDK Overview

**Agent SDK** (OpenAI's open-source framework, now widely adopted) represents a **paradigm shift** in how you build agents:

```
LangGraph philosophy:    "Build a graph of nodes and edges"
Agent SDK philosophy:    "Define agents and give them tools"
```

Agent SDK has three core primitives — and only three:

```
┌─────────────────────────────────────────┐
│                                         │
│   1. AGENTS    — the thinking entities  │
│   2. TOOLS     — what agents can do     │
│   3. HANDOFFS  — how agents delegate    │
│                                         │
└─────────────────────────────────────────┘
```

That's it. Everything else is built from these three concepts.

### Minimal Agent in Agent SDK:
```python
from agents import Agent, Runner

agent = Agent(
    name="Research Assistant",
    instructions="You are a helpful research assistant. 
                  Search for information and summarise clearly.",
    tools=[web_search, get_stock_price]
)

result = Runner.run_sync(agent, "What is INFY's current stock price?")
print(result.final_output)
```

```
Compare to LangGraph:
LangGraph version → 25+ lines of graph definition
Agent SDK version → 8 lines total

Same capability. 3x less code.
```

---

## 3. Agent Lifecycle

Every agent in Agent SDK goes through a well-defined lifecycle:

```
┌─────────────────────────────────────────────────────┐
│                   AGENT LIFECYCLE                    │
│                                                      │
│  1. INITIALISE                                       │
│     → Agent created with instructions + tools        │
│     → System prompt constructed                      │
│                                                      │
│  2. RECEIVE INPUT                                    │
│     → User message arrives                           │
│     → Added to conversation thread                   │
│                                                      │
│  3. THINK (LLM Call)                                 │
│     → LLM sees: system prompt + thread + tools       │
│     → Decides: respond directly OR call a tool       │
│                                                      │
│  4a. TOOL EXECUTION (if tool called)                 │
│     → Tool runs with LLM-provided arguments          │
│     → Result appended to thread                      │
│     → Go back to step 3 (THINK again)                │
│                                                      │
│  4b. FINAL RESPONSE (if no tool called)              │
│     → LLM generates natural language response        │
│     → Lifecycle ends                                 │
│                                                      │
│  5. RETURN OUTPUT                                    │
│     → Final output returned to caller                │
│     → Thread preserved for next turn                 │
└─────────────────────────────────────────────────────┘
```

### Lifecycle as code flow:
```
Runner.run(agent, input)
    ↓
agent.instructions → system prompt
    ↓
LLM call → tool_call OR text response?
    ↓                        ↓
tool_call              text response
    ↓                        ↓
execute tool           DONE → return
    ↓
append result to thread
    ↓
LLM call again
    ↓
(repeat until text response)
```

---

## 4. Tool Calling Patterns

Agent SDK makes tool definition extremely clean using Python decorators and type hints.

### Basic Tool:
```python
from agents import function_tool

@function_tool
def get_weather(city: str) -> str:
    """Get current weather for a city."""
    # actual weather API call
    return f"Mumbai: 32°C, Humid, Partly cloudy"
```

The SDK automatically:
- Extracts the function name → tool name
- Uses the docstring → tool description (shown to LLM)
- Uses type hints → parameter schema (tells LLM what to pass)

> **The docstring is critical.** The LLM reads it to decide when and how to use the tool. A bad docstring = tool never gets called correctly.

### Tool with Complex Parameters:
```python
from pydantic import BaseModel

class StockQuery(BaseModel):
    ticker: str
    exchange: str = "NSE"   # default value
    include_history: bool = False

@function_tool
def get_stock_data(query: StockQuery) -> dict:
    """
    Fetch stock price and optionally historical data.
    Use exchange='NSE' for Indian stocks, 'NYSE' for US stocks.
    Set include_history=True only when user asks for trends.
    """
    return {
        "ticker": query.ticker,
        "price": 1847.30,
        "exchange": query.exchange
    }
```

### Tool with Error Handling:
```python
@function_tool
def send_email(to: str, subject: str, body: str) -> str:
    """Send an email. Use only when explicitly asked to send."""
    try:
        # email sending logic
        return f"Email sent to {to} successfully"
    except Exception as e:
        return f"Failed to send email: {str(e)}"
        # Agent SDK feeds this error back to LLM
        # LLM decides how to handle it gracefully
```

---

## 5. Agent Orchestration

This is where Agent SDK truly shines over LangGraph — **multiple agents working together** with minimal setup.

### The Core Pattern:

```
Instead of one agent trying to do everything,
you build a team of specialised agents:

┌─────────────────────────────────────────┐
│          ORCHESTRATOR AGENT             │
│  "I understand the goal and delegate"   │
│                                         │
│   ┌──────────┐  ┌──────────┐           │
│   │ Research │  │ Writing  │           │
│   │  Agent   │  │  Agent   │           │
│   └──────────┘  └──────────┘           │
│   ┌──────────┐  ┌──────────┐           │
│   │  Data    │  │ Review   │           │
│   │  Agent   │  │  Agent   │           │
│   └──────────┘  └──────────┘           │
└─────────────────────────────────────────┘
```

Each agent is:
- An expert in its domain
- Unaware of the other agents (by design)
- Focused on one responsibility

---

## 6. Handoffs — Agent as a Tool

This is the most elegant concept in Agent SDK.

**The core idea:** An agent can be treated as a **tool** that another agent calls.

```python
# Specialist agents
research_agent = Agent(
    name="Researcher",
    instructions="""You are a research specialist. 
                    Search for accurate, up-to-date information.
                    Return structured findings.""",
    tools=[web_search, arxiv_search]
)

writing_agent = Agent(
    name="Writer", 
    instructions="""You are a professional writer.
                    Transform research findings into 
                    clear, engaging content.""",
    tools=[grammar_check]
)

# Orchestrator treats specialists as tools via handoff
from agents import handoff

orchestrator = Agent(
    name="Orchestrator",
    instructions="""You manage a content creation pipeline.
                    First delegate research, then delegate writing.
                    Combine outputs into a final deliverable.""",
    handoffs=[
        handoff(research_agent),   # ← agent as a tool
        handoff(writing_agent)     # ← agent as a tool
    ]
)

result = Runner.run_sync(
    orchestrator,
    "Write a 500-word article on the latest advances in RAG"
)
```

```
Execution flow:

Orchestrator: "I need research first"
    ↓
Handoff → Research Agent activates
    ↓
Research Agent: searches web, arxiv, compiles findings
    ↓
Returns findings to Orchestrator
    ↓
Orchestrator: "Now I need this written up"
    ↓
Handoff → Writing Agent activates
    ↓
Writing Agent: transforms findings into article
    ↓
Returns article to Orchestrator
    ↓
Orchestrator: combines, quality checks, returns final output ✅
```

### One-way vs Two-way Handoffs:
```python
# One-way handoff — control passes completely to specialist
# Original agent is done, specialist takes over
handoff(specialist_agent)

# Two-way handoff — specialist completes task, 
# returns control to orchestrator
handoff(specialist_agent, tool_name_override="research_topic")
# Orchestrator receives result and continues its own flow
```

---

## 7. SDK vs Framework Comparison

Now you've seen both. Here's the honest comparison:

| Dimension | LangGraph | Agent SDK |
|---|---|---|
| **Mental model** | Graph of nodes/edges | Agents + tools + handoffs |
| **Lines of code** | High | Low |
| **Learning curve** | Steep | Gentle |
| **Flexibility** | Maximum | High |
| **Multi-agent** | Painful | First-class |
| **Built-in tracing** | Via LangSmith (separate) | Built-in |
| **Production ready** | Yes | Yes |
| **Custom control flow** | Excellent | Good |
| **Best for** | Complex custom workflows | Standard agent patterns |

### When to choose which:
```
Choose Agent SDK when:
  ✅ Building multi-agent systems quickly
  ✅ Standard patterns: research, analysis, generation
  ✅ Team needs to move fast
  ✅ You want built-in observability
  ✅ Handoffs between specialists is the core pattern

Choose LangGraph when:
  ✅ You need precise control over every state transition
  ✅ Complex custom branching logic
  ✅ Non-standard agent topologies
  ✅ You're already deep in the LangChain ecosystem
  ✅ Fine-grained checkpointing requirements
```

> In the real world: **start with Agent SDK**, reach for LangGraph only when you hit its limits.

---

## 8. Assignment 04 Context — Website Automation Agent

Your syllabus mentions **Assignment 04: Website Automation Agent** for internship-exempted students. Let's connect what you've learned to what that assignment likely involves:

```python
# A website automation agent would look something like this:

from agents import Agent, Runner, function_tool
from playwright.async_api import async_playwright

@function_tool
async def navigate_to_url(url: str) -> str:
    """Navigate browser to a specified URL."""
    # playwright browser automation
    return f"Navigated to {url}"

@function_tool  
async def click_element(selector: str) -> str:
    """Click an element on the current page."""
    return f"Clicked {selector}"

@function_tool
async def fill_form_field(selector: str, value: str) -> str:
    """Fill a form field with a value."""
    return f"Filled {selector} with {value}"

@function_tool
async def extract_page_content() -> str:
    """Extract all visible text from the current page."""
    return "Page content here..."

web_automation_agent = Agent(
    name="Web Automation Agent",
    instructions="""You automate web browser interactions.
                    Break complex tasks into atomic browser actions.
                    Always verify actions succeeded before proceeding.
                    Handle errors gracefully.""",
    tools=[navigate_to_url, click_element, 
           fill_form_field, extract_page_content]
)
```

---

## How Sessions 6 & 7 Connect

```
Session 6 — LangChain & LangGraph:
  → LangChain: standardised primitives (prompts, chains, parsers)
  → LangGraph: full control via state machines
  → Great for custom complex workflows
  → Verbose, graph-centric thinking required

Session 7 — Agent SDK:
  → Higher abstraction layer
  → Agents + tools + handoffs = everything you need
  → Multi-agent orchestration is first-class
  → Built on the same LLM tool-calling primitives
  → Less code, faster iteration, built-in observability

Both solve the same problem from different angles.
LangGraph = maximum control
Agent SDK = maximum velocity
```

---

## The Agent Stack So Far

```
Layer 1 — Foundation (Session 1)
  LLM: next-token predictor
  
Layer 2 — Steering (Session 2)
  Prompt Engineering: control the prediction
  
Layer 3 — Action (Session 3)
  Agents: LLM + tools + loop
  
Layer 4 — Knowledge (Sessions 4 & 5)
  RAG: give agents memory and facts
  
Layer 5 — Orchestration (Session 6)
  LangChain/LangGraph: wire it all together
  
Layer 6 — Production Agents (Session 7)
  Agent SDK: build real multi-agent systems fast
```

---

## Quick Recap — Session 7

| Concept | One-liner |
|---|---|
| LangGraph flaws | Too verbose, graph model leaks into logic, weak multi-agent |
| Agent SDK | Agents + tools + handoffs — three primitives only |
| Agent lifecycle | Initialise → receive → think → tool/respond → return |
| Tool definition | Decorator + type hints + docstring = complete tool |
| Docstring importance | LLM reads it to decide when/how to use the tool |
| Agent orchestration | Team of specialists > one generalist |
| Handoff | Agent treated as a tool another agent can call |
| One-way handoff | Control transfers completely to specialist |
| Two-way handoff | Specialist completes and returns to orchestrator |
| SDK vs LangGraph | SDK for velocity, LangGraph for control |

---

Session 8 goes deeper into **Threads & Tracing** — how agents maintain conversation context across turns, how you observe what's happening inside an agent's execution, and how you debug when things go wrong. Ready?