# GenAI — Session 8: Threads & Tracing in Agent SDK

---

## 1. Threads in Agents

### What is a Thread?

A **thread** is the complete, ordered history of a conversation between a user and an agent — every message, every tool call, every tool result, stitched together in sequence.

> Think of a thread the same way you think of a WhatsApp chat — it's the full record of everything that was said, in order, by everyone involved.

```
Thread for "Research INFY stock" conversation:

[1] USER:      "Research Infosys and tell me if I should invest"
[2] ASSISTANT: <decides to call get_stock_price>
[3] TOOL CALL: get_stock_price(ticker="INFY")
[4] TOOL RESULT: "INFY: ₹1,847, -0.8%, P/E: 24.3"
[5] ASSISTANT: <decides to call get_news>
[6] TOOL CALL: get_news(company="Infosys")
[7] TOOL RESULT: "Infosys wins $2B Siemens contract..."
[8] ASSISTANT: "Based on current data, INFY is trading at..."
```

**The thread IS the agent's working memory for that conversation.**

Every time the agent thinks, it sees the entire thread — all previous messages, all tool calls, all results. This is how it knows what it has already done and what to do next.

---

### Why Threads Matter

Without threads:

```
Turn 1 — User: "Research Infosys"
Agent: researches, returns findings ✅

Turn 2 — User: "Now compare it with TCS"
Agent: "I'm sorry, who is Infosys?" ❌
→ Complete amnesia. No context from Turn 1.
```

With threads:

```
Turn 1 — User: "Research Infosys"
Agent: researches, appends everything to Thread #A ✅

Turn 2 — User: "Now compare it with TCS"  
Agent reads Thread #A → sees all previous research
Agent: "Continuing from my earlier analysis of Infosys..." ✅
```

---

### Thread Structure in Agent SDK

```python
from agents import Agent, Runner, Thread

# Create a persistent thread
thread = Thread()

agent = Agent(
    name="Financial Analyst",
    instructions="You are a financial analyst assistant.",
    tools=[get_stock_price, get_news, get_financials]
)

# Turn 1 — thread starts empty
result1 = Runner.run_sync(
    agent,
    "Research Infosys",
    thread=thread          # ← pass thread in
)

print(result1.final_output)
# → "Infosys is currently trading at ₹1,847..."

# Turn 2 — thread now has Turn 1's full context
result2 = Runner.run_sync(
    agent,
    "Compare it with TCS now",
    thread=thread          # ← same thread, now has Turn 1 history
)

print(result2.final_output)
# → "Comparing with my earlier analysis of Infosys..." ✅
```

---

### Thread ID — Connecting Threads to Users

In production, every user gets their own thread, identified by a **thread ID**:

```python
# User logs in → retrieve or create their thread
user_id = "user_rahul_123"

thread = Thread(thread_id=f"financial_agent_{user_id}")

# All of Rahul's conversations with this agent
# are stored under this thread ID

# Rahul closes app, comes back tomorrow:
thread = Thread(thread_id=f"financial_agent_{user_id}")
# Agent remembers everything from all previous sessions ✅
```

---

### What Lives Inside a Thread?

```
Thread
├── thread_id: "financial_agent_user_rahul_123"
├── created_at: "2025-06-01T10:30:00"
├── messages: [
│   ├── {role: "user",      content: "Research Infosys"}
│   ├── {role: "assistant", content: null,
│   │    tool_calls: [{name: "get_stock_price", args: {ticker: "INFY"}}]}
│   ├── {role: "tool",      content: "INFY: ₹1,847..."}
│   ├── {role: "assistant", content: null,
│   │    tool_calls: [{name: "get_news", args: {company: "Infosys"}}]}
│   ├── {role: "tool",      content: "Infosys wins $2B contract..."}
│   └── {role: "assistant", content: "Based on current data..."}
│   ]
└── metadata: {agent: "Financial Analyst", turns: 1}
```

Every single step — including intermediate tool calls — is preserved. This is what makes debugging possible.

---

## 2. Execution Tracing

### What is a Trace?

A **trace** is a detailed, structured record of **everything that happened** during one agent run — with timing, inputs, outputs, and costs at every step.

```
Thread  = WHAT was said (the conversation)
Trace   = HOW the agent processed it (the execution)
```

```
Trace for "Research Infosys" run:

RUN #a3f9b2
├── Total time: 4.2 seconds
├── Total tokens: 2,847
├── Total cost: $0.0043
│
├── STEP 1: LLM Call [0.0s → 0.8s]
│   ├── Input tokens: 412
│   ├── Output: tool_call → get_stock_price(INFY)
│   └── Duration: 800ms
│
├── STEP 2: Tool Execution [0.8s → 1.1s]
│   ├── Tool: get_stock_price
│   ├── Args: {"ticker": "INFY"}
│   ├── Result: "INFY: ₹1,847, -0.8%"
│   └── Duration: 300ms
│
├── STEP 3: LLM Call [1.1s → 2.3s]
│   ├── Input tokens: 891
│   ├── Output: tool_call → get_news(Infosys)
│   └── Duration: 1200ms
│
├── STEP 4: Tool Execution [2.3s → 3.1s]
│   ├── Tool: get_news
│   ├── Args: {"company": "Infosys"}
│   ├── Result: "Infosys wins $2B contract..."
│   └── Duration: 800ms
│
└── STEP 5: LLM Call [3.1s → 4.2s]
    ├── Input tokens: 1544
    ├── Output: "Based on current data, INFY..."
    └── Duration: 1100ms
```

---

### Enabling Tracing in Agent SDK

```python
from agents import Agent, Runner
from agents.tracing import TracingConfig

agent = Agent(
    name="Financial Analyst",
    instructions="You are a financial analyst.",
    tools=[get_stock_price, get_news]
)

# Run with tracing enabled
result = Runner.run_sync(
    agent,
    "Research Infosys",
    run_config=TracingConfig(
        workflow_name="financial_analysis",
        trace_id="run_001",           # optional custom ID
        metadata={"user": "rahul"}    # optional custom metadata
    )
)

# Access the trace
trace = result.trace
print(trace.total_duration_ms)    # → 4200
print(trace.total_tokens)         # → 2847
print(len(trace.steps))           # → 5
```

---

### Trace Visualisation

In production, traces are viewed in a dashboard (like OpenAI's platform or self-hosted tools like Langfuse):

```
Timeline View:

0ms    800ms   1100ms  2300ms  3100ms  4200ms
│       │        │       │       │       │
├───────┤        │       │       │       │
│LLM #1│        │       │       │       │
         ├──────┤        │       │       │
         │Tool 1│        │       │       │
                 ├───────┤       │       │
                 │LLM #2 │       │       │
                          ├──────┤       │
                          │Tool 2│       │
                                  ├──────┤
                                  │LLM #3│
```

You immediately see:
- Which step took the longest (bottleneck)
- Where errors occurred
- Total cost of the run
- Whether tools returned expected results

---

## 3. Debugging Agent Flows

Debugging agents is fundamentally different from debugging regular code.

### Why Agent Debugging is Hard

```
Regular code bug:
  Line 47: x = calculate(y)  ← deterministic
  → Same input always produces same output
  → Easy to reproduce, easy to fix

Agent bug:
  Step 3: LLM decided to call wrong tool  ← non-deterministic
  → Different input phrasing → different tool choice
  → Might not reproduce consistently
  → Hard to isolate the cause
```

### The 5 Layers of Agent Debugging

**Layer 1 — Did the right tools get called?**
```
Check trace:
  Expected: get_stock_price → get_news → respond
  Actual:   get_stock_price → respond (skipped get_news)
  
Root cause: System prompt didn't make it clear when 
            to call get_news. Fix the instructions.
```

**Layer 2 — Were tools called with correct arguments?**
```
Check trace:
  Expected: get_stock_price(ticker="INFY")
  Actual:   get_stock_price(ticker="Infosys Ltd.")
  
Root cause: Tool docstring didn't specify to use 
            ticker symbols, not company names.
            Fix: "Always use NSE ticker format e.g. INFY not Infosys"
```

**Layer 3 — Did tools return expected results?**
```
Check trace:
  Tool call: get_stock_price(ticker="INFY") ✅
  Tool result: Error: API rate limit exceeded ❌
  
Root cause: External API issue. 
  Fix: Add retry logic + fallback data source.
```

**Layer 4 — Did the LLM use tool results correctly?**
```
Check trace:
  Tool result: "INFY: ₹1,847"
  LLM output: "INFY is trading at ₹1,874" ❌ (digits transposed!)
  
Root cause: LLM hallucinated slightly despite having correct data.
  Fix: Add explicit instruction "Always quote the exact 
       number from tool results. Do not paraphrase numbers."
```

**Layer 5 — Is the final output correct?**
```
Evaluate output against ground truth.
If wrong despite correct tool calls + results:
  → System prompt issue
  → Reasoning instruction issue
  → Few-shot examples needed
```

---

### Debugging Workflow in Practice

```
Agent gives wrong output
         ↓
Open trace in dashboard
         ↓
Check: Were all expected tools called?
  NO → Fix system prompt / tool descriptions
  YES ↓
Check: Were tools called with right args?
  NO → Fix tool docstrings / add examples
  YES ↓
Check: Did tools return correct data?
  NO → Fix tool implementation / add error handling
  YES ↓
Check: Did LLM use the data correctly?
  NO → Add explicit instructions about using tool output
  YES ↓
Check: Is the system prompt causing reasoning errors?
  → Add chain-of-thought instruction
  → Add few-shot examples of correct reasoning
  → Reduce ambiguity in instructions
```

---

## 4. Observability Mindset

**Observability** is the practice of instrumenting your system so you can understand its internal state from external outputs alone.

In traditional software:
```
Logs    → what happened (events)
Metrics → how much / how fast (numbers)
Traces  → where time was spent (flow)
```

In agent systems, you need all three **plus**:
```
Prompt versions  → which prompt was active when the bug occurred
Model versions   → which model version produced the output
Token counts     → cost attribution per feature / user
Decision points  → WHY the agent chose a particular tool
Output quality   → was the answer actually correct
```

### The Observability Stack for Agents:

```
┌─────────────────────────────────────────────┐
│              OBSERVABILITY STACK             │
│                                             │
│  Agent SDK (built-in traces)                │
│       ↓                                     │
│  Tracing Backend (Langfuse / LangSmith)     │
│       ↓                                     │
│  Dashboards (latency, cost, error rate)     │
│       ↓                                     │
│  Alerts (cost spike, error rate > 5%)       │
│       ↓                                     │
│  LLM Evaluators (output quality scoring)   │
└─────────────────────────────────────────────┘
```

### Key Metrics to Track in Production:

```
Performance:
  → P50/P95/P99 latency per agent run
  → Tool execution time breakdown
  → LLM call latency vs tool latency ratio

Cost:
  → Tokens per run (input + output)
  → Cost per run by agent type
  → Cost per user / feature

Quality:
  → Tool call success rate
  → Hallucination rate (via LLM judge)
  → User satisfaction / thumbs up-down

Reliability:
  → Agent run failure rate
  → Tool failure rate by tool name
  → Retry rate (how often agents loop back)
```

---

## 5. Failure Analysis

Understanding failure modes **systematically** is what separates junior and senior AI engineers.

### Failure Taxonomy:

```
AGENT FAILURES
├── Input failures
│   ├── Ambiguous user query
│   ├── Query in unexpected language/format
│   └── Missing required context
│
├── Reasoning failures
│   ├── Wrong tool selected
│   ├── Correct tool, wrong arguments
│   ├── Premature termination (stopped too early)
│   ├── Infinite loop (never terminates)
│   └── Hallucination despite correct tool results
│
├── Tool failures
│   ├── Tool returns error (API down)
│   ├── Tool returns empty result
│   ├── Tool returns unexpected format
│   └── Tool timeout
│
└── Output failures
    ├── Correct answer, wrong format
    ├── Partially correct answer
    ├── Confident wrong answer
    └── Refused to answer (over-refusal)
```

### Building Failure Resilience:

**For Tool Failures:**
```python
@function_tool
def get_stock_price(ticker: str) -> str:
    """Get current stock price for a ticker."""
    try:
        result = primary_api.get_price(ticker)
        return f"{ticker}: ₹{result.price}"
    except APIRateLimitError:
        # Fallback to secondary source
        result = fallback_api.get_price(ticker)
        return f"{ticker}: ₹{result.price} (fallback source)"
    except Exception as e:
        # Return structured error — LLM handles it gracefully
        return f"Unable to fetch price for {ticker}. Error: {str(e)}. Try again in a moment."
```

**For Infinite Loops:**
```python
result = Runner.run_sync(
    agent,
    user_input,
    max_turns=10         # hard cap on reasoning steps
)

if result.stop_reason == "max_turns":
    # Agent hit the limit — log this as an anomaly
    logger.warning(f"Agent hit max_turns for input: {user_input}")
```

**For Reasoning Failures:**
```python
agent = Agent(
    name="Analyst",
    instructions="""
    You are a financial analyst.
    
    RULES:
    1. Always call get_stock_price BEFORE making any price claims
    2. Always call get_news if asked about recent events
    3. Never make up numbers — only use exact figures from tools
    4. If a tool fails, say so explicitly and explain what you know without it
    5. If you are uncertain, say so — do not guess
    """
)
# Explicit rules reduce reasoning failures dramatically
```

---

## Threads vs Traces — The Full Picture

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│  THREAD (the conversation)                          │
│  ─────────────────────────                         │
│  Turn 1: User → Agent → Tool → Agent → Response    │
│  Turn 2: User → Agent → Response                   │
│  Turn 3: User → Agent → Tool → Tool → Response     │
│                                                     │
│  TRACE (the execution detail for each turn)         │
│  ──────────────────────────────────────────        │
│  Turn 1 Trace:                                      │
│    LLM call #1: 800ms, 412 tokens                   │
│    Tool call:   300ms, success                      │
│    LLM call #2: 1100ms, 891 tokens                  │
│    Total: 2.2s, $0.002                              │
│                                                     │
└─────────────────────────────────────────────────────┘

Threads tell you WHAT happened.
Traces tell you HOW and WHY it happened.
Both are essential for production systems.
```

---

## How Sessions 7 & 8 Connect

```
Session 7 — Agent SDK:
  Built the machine:
  Agents + Tools + Handoffs
  
Session 8 — Threads & Tracing:
  Gave the machine a memory and a black box recorder:
  Threads  = memory across turns
  Tracing  = flight recorder for every run
  Debugging = systematic diagnosis using traces
  Observability = production health monitoring
  Failure analysis = knowing what can go wrong and why

You can build an agent in Session 7.
You can operate one reliably in Session 8.
```

---

## Quick Recap — Session 8

| Concept | One-liner |
|---|---|
| Thread | Full ordered history of a conversation — agent's working memory |
| Thread ID | Unique key connecting a thread to a user/session |
| Trace | Detailed execution record — timing, tokens, cost per step |
| Thread vs Trace | Thread = what was said; Trace = how it was processed |
| Tracing layers | LLM calls, tool calls, timing, token counts, costs |
| Debugging approach | Check tools called → args → results → LLM usage → prompt |
| Observability mindset | Instrument everything — latency, cost, quality, reliability |
| Key metrics | P95 latency, cost per run, tool success rate, hallucination rate |
| Failure taxonomy | Input, reasoning, tool, output failures |
| Failure resilience | Fallbacks, max_turns, explicit rules, structured error returns |

---

Session 9 moves to **Memory Systems in AI Agents** — the difference between an agent that forgets everything between sessions and one that genuinely learns and remembers over time. Ready?