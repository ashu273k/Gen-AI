# GenAI — Session 9: Memory Systems in AI Agents

---

## 1. Why Memory Matters

Let's start with a simple but powerful question:

**What separates a useful AI assistant from a truly intelligent one?**

```
Useful assistant:
  User: "What's the capital of France?"
  Agent: "Paris." ✅

Truly intelligent assistant:
  Session 1 — User: "I'm preparing for a trip to Paris in June"
  Session 4 — User: "What should I pack?"
  Agent: "For your Paris trip in June, you'll want light layers 
          since it averages 20°C, a good umbrella for occasional 
          rain, and comfortable walking shoes for the cobblestones." ✅
  
  → The agent remembered context from 3 sessions ago
  → Connected it to the new question
  → Gave a personalised, relevant answer
```

Without memory, every conversation starts from zero. The agent is permanently amnesiac — brilliant but forgetful.

> Memory is what transforms an agent from a **stateless tool** into a **stateful collaborator**.

---

### The Business Case for Memory

```
Without memory:
  User repeats their preferences every session     → frustrating
  Agent asks for context already provided          → annoying  
  Personalisation is impossible                    → generic
  Long-running projects lose continuity            → broken

With memory:
  Agent remembers user's name, role, preferences  → delightful
  Agent builds on previous sessions               → efficient
  Personalisation is automatic                    → valuable
  Projects have full continuity                   → reliable
```

---

## 2. Short-Term vs Long-Term Memory

This is the foundational distinction in agent memory architecture.

---

### Short-Term Memory

Short-term memory is **what the agent knows right now, within the current run.**

It lives entirely inside the **context window** — the active thread of messages the LLM can see during a single execution.

```
Context Window (Short-Term Memory):

[System Prompt]
[User: "Research Infosys"]
[Assistant: <tool_call get_stock_price>]
[Tool: "INFY: ₹1,847"]
[Assistant: <tool_call get_news>]
[Tool: "Infosys wins $2B contract"]
[Assistant: "Based on my research..."]
[User: "Compare with TCS now"]   ← current input
                                    ↑
                           Agent sees ALL of this
```

**Properties of short-term memory:**
```
Capacity:    Limited by context window (e.g. 128k tokens)
Speed:       Instantaneous — already in context
Persistence: Temporary — gone when the session ends
Cost:        Every token in context costs money per LLM call
```

**The short-term memory problem:**

As conversations grow longer, you hit two walls:

```
Wall 1 — Cost Wall:
  100 messages × 500 tokens = 50,000 tokens per LLM call
  Every single step re-processes the entire history
  → Exponentially growing cost

Wall 2 — Capacity Wall:
  Eventually the context fills up completely
  Agent starts losing the oldest messages
  → Amnesia for things said earlier in the conversation
```

---

### Long-Term Memory

Long-term memory is **what the agent knows across sessions** — information that persists beyond a single conversation.

It lives **outside the context window**, in external storage, and is selectively retrieved when needed.

```
Long-Term Memory (External Storage):

┌─────────────────────────────────────────┐
│         LONG-TERM MEMORY STORE          │
│                                         │
│  User: Rahul Sharma                     │
│  Role: Senior Product Manager           │
│  Company: Fintech startup               │
│  Preferences: Concise answers, bullet   │
│               points, no jargon         │
│  Past projects: Built payments module,  │
│                 working on fraud system  │
│  Last session: Asked about RAG for      │
│                document search          │
└─────────────────────────────────────────┘

Agent retrieves relevant facts from this store
and injects them into the context window
only when they're relevant to the current query.
```

**Properties of long-term memory:**
```
Capacity:    Unlimited (external database)
Speed:       Requires retrieval (adds latency)
Persistence: Permanent until explicitly deleted
Cost:        Storage cost + retrieval cost (much cheaper than 
             keeping everything in context)
```

---

### Short-Term vs Long-Term — Side by Side

| Dimension | Short-Term | Long-Term |
|---|---|---|
| **Location** | Context window | External database |
| **Capacity** | ~128k tokens | Unlimited |
| **Persistence** | Current session only | Across all sessions |
| **Retrieval** | Always available | Must be fetched |
| **Speed** | Instant | Adds latency |
| **Cost** | High (re-processed every step) | Low |
| **Examples** | Current conversation, recent tool results | User preferences, past project facts, domain knowledge |

---

## 3. Memory vs Context Window

This distinction trips up most beginners. Let's be precise:

```
Context Window ≠ Memory

Context Window:
  → The "RAM" of the LLM
  → Everything the model can see RIGHT NOW
  → Fixed size, set by the model (128k, 200k tokens)
  → Includes: system prompt + conversation + retrieved memory

Memory:
  → The "Hard Drive" of the agent system
  → Everything the agent has ever learned or been told
  → Unlimited size
  → Lives outside the model
  → Selectively loaded INTO the context window when needed
```

### The relationship:

```
LONG-TERM MEMORY (unlimited, external)
         ↓
    [Retrieval]  ← "What's relevant to the current query?"
         ↓
CONTEXT WINDOW (limited, active)
  ┌──────────────────────────────┐
  │ System Prompt                │
  │ Retrieved Memory Snippets    │ ← injected from long-term
  │ Current Conversation         │ ← short-term memory
  │ Tool Results                 │
  └──────────────────────────────┘
         ↓
       LLM
         ↓
     Response
```

> The context window is the **working desk**. Long-term memory is the **filing cabinet**. You can't work with everything in the cabinet at once — you pull out only the relevant folders.

---

## 4. Memory Storage Strategies

There are four distinct strategies for storing long-term memory. Each has a specific use case.

---

### Strategy 1 — In-Context Storage (Naive)

Simply append everything to the system prompt or conversation.

```python
# Store memory as text in system prompt
memory_text = """
User: Rahul Sharma, PM at fintech startup
Past conversations:
  - 2025-05-01: Asked about RAG implementation
  - 2025-05-08: Discussed vector databases
  - 2025-05-15: Asked about LangGraph agents
  - 2025-05-22: Requested deployment strategies
"""

agent = Agent(
    instructions=f"You are a helpful assistant.\n\nMemory:\n{memory_text}",
    tools=[...]
)
```

```
✅ Simple to implement
✅ LLM sees everything immediately
❌ Context grows without bound
❌ Expensive — every fact costs tokens every call
❌ Hits context limit eventually
❌ No retrieval — all or nothing
```

**Use when:** Prototype, small memory footprint, < 50 facts to remember.

---

### Strategy 2 — Vector Store Memory (Semantic Retrieval)

Store memories as **embeddings** in a vector database, retrieve relevant ones at query time.

```python
from agents import Agent
import chromadb

# Initialise vector store
memory_store = chromadb.Client()
collection = memory_store.create_collection("user_memories")

# Store a memory (after each conversation turn)
def save_memory(content: str, metadata: dict):
    embedding = embed(content)
    collection.add(
        documents=[content],
        embeddings=[embedding],
        metadatas=[metadata],
        ids=[f"mem_{timestamp()}"]
    )

# Retrieve relevant memories before each agent run
def get_relevant_memories(query: str, top_k: int = 3) -> str:
    query_embedding = embed(query)
    results = collection.query(
        query_embeddings=[query_embedding],
        n_results=top_k
    )
    return "\n".join(results["documents"][0])

# Inject memories into agent context
def run_with_memory(agent, user_query):
    relevant_memories = get_relevant_memories(user_query)
    
    augmented_input = f"""
    Relevant context from past interactions:
    {relevant_memories}
    
    Current query: {user_query}
    """
    
    return Runner.run_sync(agent, augmented_input)
```

```
Flow:
User query arrives
    ↓
Embed query → search memory vector store
    ↓
Top 3 most relevant past memories retrieved
    ↓
Injected into context alongside current query
    ↓
Agent answers with full historical context ✅
```

```
✅ Scales to millions of memories
✅ Only relevant memories loaded
✅ Semantic search — finds related concepts
❌ Adds retrieval latency
❌ Embedding + storage cost
❌ Quality depends on what got saved
```

**Use when:** Long-running assistants, user-specific knowledge, large memory footprints.

---

### Strategy 3 — Key-Value Store Memory (Structured Facts)

Store specific, structured facts as key-value pairs in a fast database (Redis, DynamoDB).

```python
import redis

memory_db = redis.Redis()

# Store structured user facts
def update_user_memory(user_id: str, facts: dict):
    for key, value in facts.items():
        memory_db.hset(f"user:{user_id}", key, value)

# Example stored facts
update_user_memory("rahul_123", {
    "name": "Rahul Sharma",
    "role": "Senior PM",
    "company": "Fintech startup",
    "preferred_format": "bullet points",
    "expertise_level": "intermediate",
    "current_project": "fraud detection system",
    "timezone": "IST"
})

# Retrieve at agent startup
def get_user_profile(user_id: str) -> dict:
    return memory_db.hgetall(f"user:{user_id}")

# Use in agent
user_profile = get_user_profile("rahul_123")

agent = Agent(
    instructions=f"""
    You are a technical assistant.
    
    User profile:
    - Name: {user_profile['name']}
    - Role: {user_profile['role']}
    - Expertise: {user_profile['expertise_level']}
    - Preferred format: {user_profile['preferred_format']}
    """,
    tools=[...]
)
```

```
✅ Extremely fast retrieval (microseconds)
✅ Precise — no ambiguity, exact values
✅ Easy to update specific facts
✅ Cheap storage
❌ No semantic search
❌ Only works for structured, known fields
❌ Can't store unstructured experiences
```

**Use when:** User preferences, profile data, settings, counters, flags.

---

### Strategy 4 — Database Memory (Episodic / Relational)

Store full conversation episodes or structured relational facts in a database (PostgreSQL, SQLite, MongoDB).

```python
# Store entire episodes (past conversations as searchable records)
CREATE TABLE memory_episodes (
    id          UUID PRIMARY KEY,
    user_id     VARCHAR(100),
    session_id  VARCHAR(100),
    timestamp   TIMESTAMP,
    summary     TEXT,           -- compressed summary of the session
    key_facts   JSONB,          -- extracted entities and facts
    full_thread JSONB           -- complete conversation if needed
);

# After each session, extract and store:
INSERT INTO memory_episodes VALUES (
    gen_random_uuid(),
    'rahul_123',
    'session_2025_06_01',
    NOW(),
    'User asked about RAG implementation for their fraud 
     detection system. Discussed chunking strategies.',
    '{"topic": "RAG", "project": "fraud detection", 
      "concepts_covered": ["chunking", "vector DB"]}',
    '{...full thread...}'
);

# At next session — retrieve relevant episodes
SELECT summary, key_facts 
FROM memory_episodes 
WHERE user_id = 'rahul_123'
ORDER BY timestamp DESC 
LIMIT 5;
```

```
✅ Full history preserved
✅ Complex queries possible (by date, topic, project)
✅ Relational — can link memories together
✅ Scalable
❌ Requires more setup
❌ No semantic search (combine with vector store for best of both)
```

**Use when:** Enterprise applications, auditable systems, complex multi-session projects.

---

### The Memory Stack — Using All Four Together

Production systems often **combine strategies**:

```
Query arrives
     ↓
┌────────────────────────────────────────┐
│         MEMORY RETRIEVAL STACK         │
│                                        │
│  1. Key-Value Store                    │
│     → Always load: user profile,       │
│       preferences, current project     │
│       (fast, always relevant)          │
│                                        │
│  2. Vector Store                       │
│     → Semantically retrieve: past      │
│       conversations, domain knowledge  │
│       (relevant to THIS query)         │
│                                        │
│  3. Database                           │
│     → Load: recent episode summaries   │
│       from last N sessions             │
│       (temporal continuity)            │
└────────────────────────────────────────┘
     ↓
Compose into context window:
  [System Prompt]
  [User Profile from KV store]
  [Relevant past facts from Vector store]
  [Recent session summaries from DB]
  [Current conversation]
     ↓
   LLM generates personalised, 
   contextually aware response ✅
```

---

### Memory Writing — When and What to Save

Retrieval is only half the problem. **You also need to decide what to save and when.**

```python
# After each agent run — extract and save memories

@function_tool
def save_to_memory(
    fact: str,
    memory_type: str,   # "user_preference" | "project_fact" | "learned_concept"
    importance: int     # 1-5 scale
) -> str:
    """
    Save an important fact to long-term memory.
    Use when user states a preference, shares important context,
    or when a key fact should be remembered for future sessions.
    """
    memory_store.save(fact, memory_type, importance)
    return f"Saved to memory: {fact}"

# The AGENT itself decides what to remember
agent = Agent(
    instructions="""
    You are a helpful assistant with memory capabilities.
    
    When users share:
    - Personal preferences → save with type "user_preference"
    - Project details → save with type "project_fact"  
    - Important decisions made → save with type "project_fact"
    
    Do NOT save:
    - Trivial small talk
    - Information that changes frequently
    - Sensitive personal data
    """,
    tools=[save_to_memory, get_relevant_memories, ...]
)
```

---

## Putting It All Together — The Memory-Enabled Agent

```
COMPLETE MEMORY ARCHITECTURE:

┌─────────────────────────────────────────────────────┐
│                  AGENT RUN FLOW                      │
│                                                      │
│  1. User sends message                               │
│          ↓                                           │
│  2. Load user profile (KV store) — always            │
│          ↓                                           │
│  3. Retrieve relevant memories (Vector store)        │
│          ↓                                           │
│  4. Load recent session summaries (DB)               │
│          ↓                                           │
│  5. Compose context window:                          │
│     [profile + memories + summaries + conversation]  │
│          ↓                                           │
│  6. Agent thinks + uses tools + responds             │
│          ↓                                           │
│  7. Extract memorable facts from this session        │
│          ↓                                           │
│  8. Save to appropriate memory store                 │
│          ↓                                           │
│  9. Compress conversation → episode summary          │
│          ↓                                           │
│  10. Store summary in DB for next session            │
└─────────────────────────────────────────────────────┘
```

---

## How Sessions 8 & 9 Connect

```
Session 8 — Threads & Tracing:
  Threads = short-term memory within a session
  Traces  = execution record of each run

Session 9 — Memory Systems:
  Goes beyond threads →
  How to make agents remember across sessions
  Four storage strategies for different needs
  Memory retrieval = RAG applied to agent history
  
Key insight:
  Long-term memory retrieval uses the SAME
  vector search principles as RAG (Session 4)
  applied to the agent's own past experiences
  instead of external documents.
```

---

## Quick Recap — Session 9

| Concept | One-liner |
|---|---|
| Why memory matters | Transforms stateless tool into stateful collaborator |
| Short-term memory | Lives in context window — current session only |
| Long-term memory | Lives in external storage — persists across sessions |
| Memory vs context window | Filing cabinet vs working desk |
| In-context storage | Simplest — append facts to prompt, doesn't scale |
| Vector store memory | Semantic retrieval — finds relevant past experiences |
| Key-value store | Fast structured facts — profiles, preferences, settings |
| Database memory | Full episodic history — complex queries, audit trails |
| Memory stack | Combine all four for production systems |
| Memory writing | Agent decides what to save — preferences, facts, decisions |
| Session compression | Summarise conversations into episodes after each session |

---

Session 10 dives into **Graph Databases & Relational Memory** — a completely different way of storing knowledge that captures not just facts but the **relationships between facts**, using Neo4j and knowledge graphs. Ready?