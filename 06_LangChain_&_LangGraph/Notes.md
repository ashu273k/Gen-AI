# GenAI — Session 6: LangChain Fundamentals & LangGraph — Orchestration & Stateful Agents

---

## 1. Why Frameworks Are Required

By now you've learned:
- LLMs predict tokens
- Prompts steer them
- Agents use tools in a loop
- RAG retrieves relevant context

Now imagine building a production system that does **all of this together**:

```
Without a framework — you write:
  ✗ Manual prompt formatting every time
  ✗ Custom retry logic for API failures
  ✗ Your own tool-calling parser
  ✗ Manual memory management
  ✗ Custom chain orchestration
  ✗ Your own streaming handler
  ✗ Custom output parsers
  ✗ State management from scratch

→ 2000+ lines of boilerplate before any business logic
```

Frameworks like **LangChain** solve this by providing standardised, composable building blocks so you focus on **what** your system does, not **how** to wire it together.

> LangChain is to LLM applications what Express.js is to web servers — it doesn't do the work for you, but it removes the scaffolding problem.

---

## 2. LangChain Architecture

LangChain is built around four core primitives. Master these and you master LangChain.

---

### Primitive 1 — PromptTemplates

Instead of hardcoding prompts as strings, PromptTemplates are **reusable, parameterised prompt factories**.

```python
# Without LangChain — messy, error-prone
query = "climate change"
prompt = f"Write a summary about {query} for a 10 year old."

# With LangChain PromptTemplate — clean, reusable
from langchain.prompts import PromptTemplate

template = PromptTemplate(
    input_variables=["topic", "audience"],
    template="Write a summary about {topic} for a {audience}."
)

prompt = template.format(topic="climate change", audience="10 year old")
```

Templates can also handle:
- Few-shot examples
- System + user message separation (ChatPromptTemplate)
- Dynamic example selection

---

### Primitive 2 — Runnables

Everything in modern LangChain is a **Runnable** — an object with a standard interface:

```python
runnable.invoke(input)     # single input → single output
runnable.batch(inputs)     # list of inputs → list of outputs
runnable.stream(input)     # input → streaming output tokens
```

This standardisation means **any component can connect to any other component** — models, prompts, retrievers, tools, parsers all speak the same language.

---

### Primitive 3 — Chains

Chains are **sequences of Runnables** connected together into a pipeline.

```python
# LCEL (LangChain Expression Language) — uses | pipe operator
chain = prompt_template | llm | output_parser

# This reads as:
# 1. Format the prompt
# 2. Send to LLM
# 3. Parse the output

result = chain.invoke({"topic": "climate change", "audience": "10 year old"})
```

### A RAG Chain looks like this:

```python
rag_chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | prompt_template
    | llm
    | StrOutputParser()
)

answer = rag_chain.invoke("What is the leave policy?")
```

```
Flow:
User question
    ↓
Retriever fetches relevant chunks → "context"
Question passes through unchanged → "question"
    ↓
PromptTemplate formats both into a prompt
    ↓
LLM generates response
    ↓
StrOutputParser extracts clean text
    ↓
Final answer ✅
```

---

### Primitive 4 — Output Parsers

LLM outputs are raw strings. Output Parsers **structure them** into usable Python objects.

```python
from langchain.output_parsers import JsonOutputParser
from pydantic import BaseModel

class MovieReview(BaseModel):
    title: str
    rating: int
    summary: str

parser = JsonOutputParser(pydantic_object=MovieReview)

# Parser adds formatting instructions to your prompt automatically
# AND converts the LLM's JSON string → Python object

review = chain.invoke("Review Inception")
print(review.rating)  # → 9  (actual int, not a string)
```

---

## 3. LCEL — LangChain Expression Language

LCEL is LangChain's **composition syntax** using the `|` pipe operator. It's the modern way to build chains.

```python
# The pipe operator chains Runnables left to right
chain = A | B | C

# Equivalent to:
output_A = A.invoke(input)
output_B = B.invoke(output_A)
output_C = C.invoke(output_B)
```

### Why LCEL matters:
```
✅ Automatic streaming support
✅ Automatic async support  
✅ Built-in retry logic
✅ Parallel execution where possible
✅ Easy to inspect intermediate steps
✅ Works with LangSmith for tracing
```

### Parallel execution in LCEL:
```python
# Run two retrievers simultaneously, merge results
from langchain.schema.runnable import RunnableParallel

parallel_chain = RunnableParallel(
    answer=rag_chain,
    sources=source_chain
)
# Both run at the same time → faster than sequential
```

---

## 4. Tool Calling Basics in LangChain

Tools are **functions the LLM can decide to call**. LangChain standardises how you define and bind them.

```python
from langchain.tools import tool

@tool
def get_stock_price(ticker: str) -> str:
    """Get the current stock price for a given ticker symbol."""
    # actual API call here
    return f"{ticker}: ₹1,847"

@tool  
def send_email(to: str, subject: str, body: str) -> str:
    """Send an email to a specified address."""
    # actual email sending here
    return "Email sent successfully"

# Bind tools to the LLM
llm_with_tools = llm.bind_tools([get_stock_price, send_email])
```

```
Flow when user asks "What's INFY price?":

LLM sees tools available
    ↓
LLM decides: "I should call get_stock_price"
    ↓
Returns: {"tool": "get_stock_price", "args": {"ticker": "INFY"}}
    ↓
LangChain executes the actual function
    ↓
Result fed back to LLM
    ↓
LLM generates final natural language response
```

---

## 5. What is LangGraph?

Here's where things get significantly more powerful.

**LangChain chains are linear** — A → B → C → Done.

But real agent workflows are **not linear**. They have:
- Loops (retry if output isn't good enough)
- Branches (if X do Y, else do Z)
- Parallel paths
- Human-in-the-loop pauses
- State that persists across steps

**LangGraph** is LangChain's answer to this — it models agent workflows as **state machines**.

> If LangChain chains are like a straight road, LangGraph is like a road network — with intersections, loops, and multiple possible routes.

---

## 6. Core LangGraph Concepts

### State
The **shared memory** of the entire graph — a dictionary that every node can read and write.

```python
from typing import TypedDict, List

class AgentState(TypedDict):
    messages: List[str]      # conversation history
    retrieved_docs: List[str] # RAG results
    tool_calls: List[dict]   # tools invoked so far
    final_answer: str        # output
    iteration_count: int     # loop counter
```

Every node receives the full state, modifies what it needs, and returns the updated state.

---

### Nodes
Nodes are **functions that do work** — they receive state, process it, and return updated state.

```python
def retrieve_node(state: AgentState) -> AgentState:
    query = state["messages"][-1]
    docs = retriever.invoke(query)
    return {"retrieved_docs": docs}  # only update what changed

def generate_node(state: AgentState) -> AgentState:
    answer = llm.invoke(
        f"Context: {state['retrieved_docs']}\n"
        f"Question: {state['messages'][-1]}"
    )
    return {"final_answer": answer.content}

def grade_node(state: AgentState) -> AgentState:
    # LLM judge checks answer quality
    score = judge_llm.invoke(f"Is this answer grounded? {state['final_answer']}")
    return {"grade": score}
```

---

### Edges
Edges define **how the graph flows** between nodes.

**Normal edges** — always go from A to B:
```python
graph.add_edge("retrieve", "generate")
# After retrieve, always go to generate
```

**Conditional edges** — branch based on state:
```python
def should_retry(state: AgentState) -> str:
    if state["grade"] == "poor" and state["iteration_count"] < 3:
        return "retry"      # → go back to retrieve node
    else:
        return "finish"     # → go to END

graph.add_conditional_edges(
    "grade_node",
    should_retry,
    {"retry": "retrieve", "finish": END}
)
```

---

## 7. Building a LangGraph — Full Example

Let's build the **Corrective RAG** system from Session 5 using LangGraph:

```
Graph Structure:

START
  ↓
retrieve_node
  ↓
grade_node ──── poor quality + retries left ──→ rewrite_node
  |                                                    ↓
  | good quality                                  retrieve_node
  ↓
generate_node
  ↓
END
```

```python
from langgraph.graph import StateGraph, END

# 1. Define the graph
workflow = StateGraph(AgentState)

# 2. Add nodes
workflow.add_node("retrieve", retrieve_node)
workflow.add_node("grade", grade_node)
workflow.add_node("rewrite", rewrite_query_node)
workflow.add_node("generate", generate_node)

# 3. Add edges
workflow.set_entry_point("retrieve")
workflow.add_edge("retrieve", "grade")
workflow.add_conditional_edges(
    "grade",
    should_retry,
    {"retry": "rewrite", "finish": "generate"}
)
workflow.add_edge("rewrite", "retrieve")  # loop back
workflow.add_edge("generate", END)

# 4. Compile
app = workflow.compile()

# 5. Run
result = app.invoke({
    "messages": ["What is the sick leave policy?"],
    "iteration_count": 0
})
```

---

## 8. Multi-Step Reasoning Flows

LangGraph shines for **complex reasoning** that requires multiple thinking steps:

```
Example: Research Agent

START
  ↓
plan_node          → "I need to: 1) search topic, 2) find examples, 3) summarise"
  ↓
search_node        → calls web search tool
  ↓
evaluate_node      → "Do I have enough information?"
  ↓                   NO ──→ search_node (loop)
  ↓ YES
synthesise_node    → combines all gathered info
  ↓
format_node        → structures into final report
  ↓
END
```

Each node does **one focused thing**. The graph manages the flow. This separation of concerns makes complex agents debuggable.

---

## 9. Tool Orchestration in Graphs

In LangGraph, tools are called **within nodes** but the graph decides **when** to call them:

```python
def tool_node(state: AgentState) -> AgentState:
    # Execute whatever tool the LLM decided to call
    last_message = state["messages"][-1]
    tool_name = last_message.tool_calls[0]["name"]
    tool_args = last_message.tool_calls[0]["args"]
    
    # Route to correct tool
    result = tools_map[tool_name].invoke(tool_args)
    
    return {"messages": state["messages"] + [result]}

def should_use_tool(state: AgentState) -> str:
    last_message = state["messages"][-1]
    if last_message.tool_calls:
        return "tool"    # → go to tool_node
    else:
        return "end"     # → go to END

graph.add_conditional_edges("llm_node", should_use_tool,
    {"tool": "tool_node", "end": END})
graph.add_edge("tool_node", "llm_node")  # loop back after tool
```

This creates the classic **ReAct loop** from Session 3, but now it's a proper graph with state.

---

## 10. Agent State Persistence & Checkpointing

One of LangGraph's most powerful production features — **saving agent state** so execution can be paused and resumed.

```python
from langgraph.checkpoint.sqlite import SqliteSaver

# Attach a checkpointer
memory = SqliteSaver.from_conn_string("agent_memory.db")
app = workflow.compile(checkpointer=memory)

# Run with a thread ID (identifies this conversation)
config = {"configurable": {"thread_id": "user_123_session_1"}}
result = app.invoke({"messages": ["Start research on RAG"]}, config)

# Hours later — resume the SAME conversation
result = app.invoke({"messages": ["Now summarise what you found"]}, config)
# Agent remembers everything from the previous run ✅
```

### Why checkpointing matters:

```
Without checkpointing:
  Long task fails at step 7 of 10 → start over from step 1
  User closes browser → all agent progress lost
  Human needs to review step 5 → can't pause

With checkpointing:
  Failure at step 7 → resume from step 7 ✅
  User returns next day → agent picks up exactly where it left off ✅
  Human review → agent pauses, waits, continues after approval ✅
```

---

## LangChain vs LangGraph — When to Use Which

```
Use LangChain (LCEL chains) when:
  ✅ Linear pipeline: prompt → LLM → parse → done
  ✅ Simple RAG: retrieve → generate
  ✅ One-shot tasks with no branching
  ✅ Quick prototypes

Use LangGraph when:
  ✅ Agent needs to loop (retry, self-correct)
  ✅ Conditional branching based on LLM output
  ✅ Multi-agent coordination
  ✅ State needs to persist across turns
  ✅ Human-in-the-loop workflows
  ✅ Complex multi-step reasoning
  ✅ Production systems that need observability
```

---

## How Everything Connects So Far

```
Session 1: LLM = token predictor
Session 2: Prompts = steer the prediction
Session 3: Agents = LLM + tools + loop
Session 4: RAG = give agents a knowledge base
Session 5: Advanced RAG = make retrieval production-grade
Session 6: LangChain = standardised building blocks
           LangGraph = orchestrate complex stateful flows

LangGraph is the "operating system" for everything you've 
learned so far — it gives RAG, agents, tools, and memory 
a proper runtime with state, branching, and persistence.
```

---

## Quick Recap — Session 6

| Concept | One-liner |
|---|---|
| Why frameworks | Remove boilerplate, standardise composition |
| PromptTemplate | Reusable parameterised prompt factory |
| Runnable | Standard interface — invoke / batch / stream |
| Chain | Sequence of Runnables connected with pipe operator |
| Output Parser | Converts LLM string output → typed Python objects |
| LCEL | Pipe-based composition syntax for chains |
| Tool calling | Bind functions to LLM so it can decide to call them |
| LangGraph | State machine framework for non-linear agent workflows |
| State | Shared dictionary all graph nodes read and write |
| Nodes | Functions that receive state, do work, return updated state |
| Edges | Define flow — normal (always) or conditional (branching) |
| Checkpointing | Persist agent state — pause, resume, human-in-the-loop |

---

Session 7 moves to the **Agent SDK** — a newer, more opinionated framework that addresses LangGraph's shortcomings for production agent systems. Ready?