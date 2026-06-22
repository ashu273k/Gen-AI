# GenAI — Session 5: RAG 2 — Advanced Retrieval Strategies

---

## 1. Bottlenecks in Basic RAG

Before learning the fixes, you need to understand exactly **where basic RAG breaks down** in production.

```
Basic RAG Pipeline:
User Query → Embed → Search → Top K Chunks → LLM → Answer
```

This works great in demos. In production, it fails in predictable ways:

### Bottleneck 1 — The Vocabulary Gap
```
User asks:  "What's the remuneration for senior engineers?"
Document says: "Salary bands for L5 and above..."

Basic RAG: No match ❌
→ "remuneration" and "salary" mean the same thing
   but their vectors aren't close enough
```

### Bottleneck 2 — Vague Queries
```
User asks: "Tell me about the policy"
→ Which policy? HR? Refund? Privacy?
→ Retrieval returns random chunks. LLM hallucinates.
```

### Bottleneck 3 — Complex Multi-Part Questions
```
User asks: "Compare the revenue of Infosys and TCS 
            in Q3 2024 and explain the difference"
→ Single query vector can't capture both sub-questions
→ Retrieval misses half the relevant chunks
```

### Bottleneck 4 — Retrieved Chunks Are Relevant But Ranked Poorly
```
Top chunk retrieved: Loosely related paragraph
Actual answer: Buried in chunk #8
→ LLM never sees the most relevant content
```

### Bottleneck 5 — Context Window Gets Polluted
```
Top K = 10 chunks retrieved
3 are highly relevant
7 are noise
→ LLM gets confused by irrelevant context
→ Answer quality drops
```

> Every advanced RAG technique you're about to learn is a **direct fix to one of these bottlenecks.**

---

## 2. The Speed vs Accuracy Tradeoff

Before adding complexity, understand the fundamental tension in RAG systems:

```
SPEED ◄─────────────────────────► ACCURACY

Fast:                              Accurate:
• Simple vector search             • Re-ranking
• Small K (fewer chunks)          • Large K + filtering  
• No query rewriting              • Query expansion
• Approximate nearest neighbour   • Exact nearest neighbour

Adding accuracy steps = adding latency.
Every technique below costs milliseconds.
```

In production you always ask: **"Is this accuracy gain worth this latency cost?"**

| Technique | Latency Cost | Accuracy Gain |
|---|---|---|
| Query rewriting | Low | High |
| Sub-query decomposition | Medium | High |
| Re-ranking | Medium | Very High |
| Corrective RAG | High | Very High |
| HyDE | Low | Medium-High |

---

## 3. Query Rewriting Using SLMs

**The fix for:** Vocabulary gap + vague queries

### The Idea:
Before embedding the user's query, use a **Small Language Model (SLM)** to rewrite it into a cleaner, more retrieval-friendly version.

```
Original query: "what's the thing about leave policy"
                           ↓
               SLM rewrites it to:
"How many annual, sick, and casual leaves are 
 employees entitled to under the company HR policy?"
                           ↓
          Now embed THIS for retrieval ✅
```

SLMs (smaller, faster models like Phi-3, Gemma-2B) are used here instead of full LLMs because:
- They're fast and cheap
- Rewriting is a simple task — no need for GPT-4 level capability
- You don't want rewriting to become the bottleneck

### Query Translation
A broader family of query rewriting techniques:

```
Technique 1 — Rephrase:
"Tell me about leaves" → "What is the employee leave policy?"

Technique 2 — Expand with synonyms:
"salary" → "salary OR remuneration OR compensation OR pay"

Technique 3 — Back-translation:
English → Hindi → back to English
(Different phrasing emerges, captures different embeddings)
```

---

## 4. LLM Judges

**The fix for:** Polluted context, poor chunk quality

### The Idea:
After retrieval, before generation, use an LLM to **evaluate whether each retrieved chunk is actually relevant** to the query.

```
Retrieved Chunks:
  Chunk 1: "Annual leave policy states 18 days per year..."
  Chunk 2: "The office cafeteria operates from 8am to 8pm..."
  Chunk 3: "Sick leave can be carried forward up to 6 days..."

LLM Judge evaluates each:
  Chunk 1: Relevant ✅
  Chunk 2: Not relevant ❌ → discard
  Chunk 3: Relevant ✅

Only Chunk 1 + 3 go into the final prompt.
```

The judge is essentially answering: *"Does this chunk contain information useful for answering the query?"*

This is also called **relevance filtering** and dramatically reduces context pollution.

> LLM judges can also be used to evaluate the **final answer** — checking if the response is actually grounded in the retrieved chunks or if the model started hallucinating.

---

## 5. Sub-Query Enhancement

**The fix for:** Complex multi-part questions

### The Idea:
Decompose a complex query into **multiple simpler sub-queries**, retrieve chunks for each independently, then combine results before generation.

```
Original: "Compare Infosys and TCS revenue in Q3 2024 
           and explain the key difference"

Decompose into:
  Sub-query 1: "Infosys Q3 2024 revenue"
  Sub-query 2: "TCS Q3 2024 revenue"  
  Sub-query 3: "Infosys TCS revenue comparison factors"

Retrieve chunks for each sub-query separately
           ↓
Merge all retrieved chunks (deduplicate)
           ↓
Feed combined context to LLM
           ↓
LLM now has everything it needs ✅
```

### Why this works:
A single query vector is a **single point** in vector space. It can only be "close to" one region of your knowledge base at a time. Multiple sub-queries = multiple search points = complete coverage.

---

## 6. Corrective RAG (CRAG)

**The fix for:** When retrieved documents are simply wrong or insufficient

### The Idea:
Add a **self-correction loop** — if retrieved chunks are evaluated as low quality, trigger a fallback (like a web search) before generating the answer.

```
User Query
    ↓
Retrieve chunks from Vector DB
    ↓
LLM Judge evaluates chunk quality
    ↓
┌───────────────────────────────┐
│ High confidence? → Generate   │ ✅ Normal RAG path
│                               │
│ Low confidence?  → CORRECT    │ 
│   → Rewrite query             │
│   → Search the web            │
│   → Use web results instead   │ ✅ Corrective path
└───────────────────────────────┘
    ↓
Final Answer (grounded either way)
```

CRAG is essentially RAG with a **safety net**. Instead of silently hallucinating when retrieval fails, it detects the failure and corrects course.

> This is the difference between a RAG system that fails gracefully vs one that confidently gives wrong answers.

---

## 7. HyDE — Hypothetical Document Embeddings

**The fix for:** Vocabulary gap between queries and documents

### The Problem it Solves:
```
User query: "Why did revenue drop?"    ← short, vague
Document:   "Q3 results declined due to reduced enterprise 
             spending and supply chain disruptions..."  ← detailed

Their embeddings are far apart.
Basic RAG misses this document. ❌
```

### The HyDE Idea:
Instead of embedding the **query**, ask the LLM to generate a **hypothetical answer** first, then embed that.

```
Step 1 — Original query:
"Why did revenue drop?"

Step 2 — LLM generates a hypothetical answer:
"Revenue may have dropped due to macroeconomic headwinds,
 reduced enterprise spending, supply chain issues, or 
 increased competition in key markets..."

Step 3 — Embed the HYPOTHETICAL ANSWER (not the query)

Step 4 — Search Vector DB with this richer vector
→ Now matches the actual document ✅
```

### Why it works:
The hypothetical answer **lives in the same "language space"** as the actual documents. It's long, detailed, and uses domain vocabulary — exactly like the documents you're searching through.

```
Query vector:              ●  (far from documents)
                          
Hypothetical answer vector:        ●  (close to documents)
                          
Actual relevant document:              ●
```

> HyDE is counterintuitive — you're using a **hallucinated answer** to find the **real answer**. But it works remarkably well in practice.

---

## 8. Re-Ranking Strategies (Cross-Encoders)

**The fix for:** Poor ranking of retrieved chunks

### Two-Stage Retrieval:

**Stage 1 — Bi-encoder (fast, approximate):**
```
Embed query → search vector DB → get Top 20 candidates
Fast: milliseconds
Approximate: might not be perfectly ranked
```

**Stage 2 — Cross-encoder re-ranker (slow, precise):**
```
Take Top 20 candidates
Feed each one JOINTLY with the query into a cross-encoder model
Cross-encoder scores each pair: (query, chunk) → relevance score
Re-rank by score → take Top 3
Slow: but much more accurate
```

### Why Cross-Encoders Are More Accurate:

```
Bi-encoder:                Cross-encoder:
Query → vector A           Query + Chunk → evaluated TOGETHER
Chunk → vector B           Model sees the interaction
Compare A and B            Much richer relevance signal
(indirect comparison)      (direct comparison)
```

Think of it this way:
- Bi-encoder is like judging a movie by its poster (fast, rough)
- Cross-encoder is like actually watching the movie (slow, accurate)

```
Final pipeline with re-ranking:

Query → Embed → Retrieve Top 20 → Re-rank → Top 3 → LLM → Answer
         fast                      precise
```

---

## 9. Context Window & Token Bottlenecks

Even with perfect retrieval, you can still hit a hard wall: **the context window**.

```
Context window = 128,000 tokens (Claude)

Your retrieved chunks: 50 chunks × 500 tokens = 25,000 tokens
Your system prompt:    2,000 tokens
Conversation history:  10,000 tokens
Your query:            100 tokens
─────────────────────────────────
Total input:           37,100 tokens ← fine

But if you retrieve 200 chunks:
200 × 500 = 100,000 tokens → Almost full context window
→ No room for the answer
→ LLM starts losing earlier context ("lost in the middle" problem)
```

### The "Lost in the Middle" Problem:
Research shows LLMs pay most attention to content at the **beginning and end** of the context. Content in the middle gets relatively ignored.

```
[STRONG ATTENTION]  ← beginning of context
...
[WEAK ATTENTION]    ← middle of context (your most relevant chunk?)
...
[STRONG ATTENTION]  ← end of context
```

**Fix:** Put your most relevant chunks at the **beginning or end** of the context, never buried in the middle.

---

## 10. Chunk Size & Overlap Tradeoffs

Two dials you tune constantly in RAG systems:

### Chunk Size:
```
Small chunks (128 tokens):
✅ Precise retrieval — very specific content
❌ Loses surrounding context
❌ More chunks = more storage + slower indexing

Large chunks (1024 tokens):
✅ Rich context — full paragraphs
❌ Less precise — retrieved chunk contains lots of irrelevant text
❌ Uses more context window per chunk
```

### Chunk Overlap:
To prevent losing context at chunk boundaries, chunks are made to **overlap**:

```
Without overlap:
Chunk 1: "...The revenue grew by 23% in Q3. [END]"
Chunk 2: "[START] This was driven by enterprise sales..."

With overlap (100 token overlap):
Chunk 1: "...The revenue grew by 23% in Q3."
Chunk 2: "The revenue grew by 23% in Q3. This was driven by enterprise sales..."
                ↑ repeated for continuity
```

### Practical Starting Points:
```
General documents:    chunk_size=512,  overlap=50
Technical docs:       chunk_size=256,  overlap=30
Legal/policy docs:    chunk_size=1024, overlap=100
Conversational data:  chunk_size=128,  overlap=20
```

> There's no universal optimal — you **tune these empirically** by measuring retrieval quality on your specific dataset.

---

## The Full Advanced RAG Pipeline

Putting it all together:

```
User Query
    ↓
① Query Rewriting (SLM) — fix vague/poor queries
    ↓
② Sub-query Decomposition — handle complex questions
    ↓
③ HyDE (optional) — generate hypothetical answer for embedding
    ↓
④ Vector Search — retrieve Top K candidates (fast, approximate)
    ↓
⑤ LLM Judge — filter irrelevant chunks
    ↓
⑥ Cross-Encoder Re-ranking — re-order by true relevance
    ↓
⑦ Top N chunks injected into prompt (position-aware)
    ↓
⑧ LLM generates grounded answer
    ↓
⑨ Corrective RAG check — if low confidence, trigger web search
    ↓
Final Answer ✅
```

Not every system uses all 9 steps. You add complexity only where your evaluation data shows it's needed.

---

## Sessions 4 → 5 Connection

```
Session 4: Basic RAG
→ Chunk → Embed → Index → Query → Retrieve → Generate
→ Works great for simple, well-phrased queries on clean docs

Session 5: Advanced RAG
→ Every technique is a targeted fix for a specific failure mode
→ Real production RAG = Basic RAG + selective advanced techniques
→ You only add complexity where you can MEASURE the improvement
```

---

## Quick Recap — Session 5

| Concept | One-liner |
|---|---|
| RAG bottlenecks | Vocab gap, vague queries, multi-part questions, poor ranking |
| Speed vs accuracy | Every accuracy improvement costs latency |
| Query rewriting | SLM cleans up query before embedding |
| Query translation | Rephrase, expand, back-translate for better coverage |
| LLM judges | Filter irrelevant chunks after retrieval |
| Sub-query enhancement | Decompose complex queries into simpler parallel searches |
| Corrective RAG | Detect retrieval failure → fallback to web search |
| HyDE | Embed a hypothetical answer instead of the query |
| Cross-encoder re-ranking | Two-stage retrieval: fast approximate → slow precise |
| Lost in the middle | LLMs ignore context buried in the middle |
| Chunk size & overlap | Tune empirically — no universal best setting |

---

Session 6 is **LangChain & LangGraph** — where you stop building RAG manually and start using frameworks that wire all of this together into production-grade pipelines. Ready?