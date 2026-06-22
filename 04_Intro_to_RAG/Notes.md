Just a quick note — in your syllabus, **RAG is Session 4**, split across two sessions (4 and 5). But no worries, let's dive straight in!

---

# GenAI — Session 4: RAG 1 — Introduction to Retrieval-Augmented Generation

---

## 1. What is RAG and Why Does It Exist?

Let's start with the problem RAG solves.

A plain LLM has three fundamental knowledge problems:

```
Problem 1 — Knowledge Cutoff
"What happened in the IPL final yesterday?"
→ LLM has no idea. Training data is frozen.

Problem 2 — No Private Data
"Summarise our company's Q3 internal report."
→ LLM never saw your company's documents.

Problem 3 — Hallucination
"What does Section 4.2 of our HR policy say?"
→ LLM makes something up confidently. Dangerous.
```

**RAG fixes all three** by giving the LLM **relevant documents at query time** — pulled from your own knowledge base.

> RAG = Give the LLM the right pages from the right book, exactly when it needs them, so it can answer accurately.

### The Core Idea in One Line:
**Retrieve** relevant context → **Augment** the prompt with it → **Generate** a grounded answer.

---

## 2. RAG vs Fine-Tuning — The Decision Framework

This is one of the most common questions in applied AI. When do you RAG, and when do you fine-tune?

### Fine-Tuning
You **retrain** the model on your custom data, baking knowledge into the model's weights permanently.

### RAG
You keep the base model unchanged and **inject relevant knowledge** into the prompt at query time.

| Dimension | RAG | Fine-Tuning |
|---|---|---|
| **Data freshness** | ✅ Update anytime | ❌ Requires retraining |
| **Cost** | Low | Very high |
| **Transparency** | ✅ Can cite sources | ❌ Black box |
| **Best for** | Factual Q&A, docs, search | Style, tone, format, domain behaviour |
| **Data size needed** | Any size | Thousands of examples minimum |
| **Hallucination risk** | Lower (grounded) | Higher |

### Decision Framework:
```
Is your data changing frequently?          → RAG
Do you need source citations?             → RAG
Do you want to change HOW the model 
  reasons or writes?                      → Fine-tune
Do you want the model to learn a 
  new skill (e.g. medical diagnosis)?     → Fine-tune
Do you have < 1000 examples?              → RAG
```

> In 90% of enterprise use cases, **RAG is the right first choice**. Fine-tuning is often overkill and expensive.

---

## 3. Vector Databases Overview

RAG requires a special kind of database — one that can search by **meaning**, not just keywords.

### Why not just use a regular database?

```
User query: "How do I reset my account password?"

Regular DB (keyword search):
→ Looks for documents containing "reset", "account", "password"
→ Misses: "I forgot my login credentials" (same meaning, different words)

Vector DB (semantic search):
→ Converts query to a vector
→ Finds documents with similar meaning, regardless of exact words
→ Finds: "I forgot my login credentials" ✅
```

### The Three Main Vector DBs in Your Syllabus:

**Chroma**
- Open-source, runs locally
- Perfect for prototyping and small projects
- No infrastructure needed — just `pip install chromadb`
- Your Assignment 03 (NotebookLM RAG) likely uses this under the hood

**Pinecone**
- Fully managed cloud service
- Production-grade, scales to billions of vectors
- Pay-as-you-go, no infrastructure management
- Go-to choice for startups shipping fast

**Qdrant**
- Open-source but also has cloud offering
- Extremely fast, supports filtering alongside vector search
- Growing favourite for production self-hosted deployments

---

## 4. Embeddings & Indexing

This is the **engine** behind RAG. Understanding embeddings is non-negotiable.

### What is an Embedding?

An embedding is a **numerical vector** that captures the *semantic meaning* of a piece of text.

```
"I love dogs"      → [0.21, -0.45, 0.87, 0.33, ...]  (384 numbers)
"I adore puppies"  → [0.22, -0.43, 0.85, 0.31, ...]  ← very close!
"The stock fell"   → [-0.67, 0.91, -0.23, 0.54, ...] ← far away
```

Texts with similar meaning produce vectors that are **close together** in high-dimensional space. This is measured using **cosine similarity**.

### The Indexing Process:

```
Your Documents (PDFs, docs, web pages)
           ↓
     Split into chunks
           ↓
  Each chunk → Embedding Model → Vector
           ↓
   Store (vector + original text) in Vector DB
           ↓
         INDEXED ✅
```

This indexing happens **once** (or when data updates). It's like building the index at the back of a textbook — expensive once, fast to look up forever.

---

## 5. Document Chunking Strategies

Before embedding, documents must be split into chunks. **How you chunk dramatically affects RAG quality.** This is underappreciated by beginners.

### Why chunk at all?
- You can't embed an entire 500-page document as one vector — you'd lose all specificity
- Smaller chunks = more precise retrieval
- But too small = lost context

### The 4 Main Strategies:

**1. Fixed-Size Chunking**
Split every N characters/tokens, regardless of content.
```
Chunk 1: "The company was founded in 1994. It grew rapidl"
Chunk 2: "y through the 2000s. Revenue hit $1B in 2010."
```
- ✅ Simple, fast
- ❌ Splits mid-sentence, loses context

**2. Sentence Chunking**
Split at sentence boundaries.
```
Chunk 1: "The company was founded in 1994."
Chunk 2: "It grew rapidly through the 2000s."
Chunk 3: "Revenue hit $1B in 2010."
```
- ✅ Preserves sentence integrity
- ❌ Sentences alone may lack context

**3. Semantic Chunking**
Group sentences that talk about the **same idea** together, using embedding similarity to detect topic shifts.
```
Chunk 1: "Founded in 1994... grew through 2000s... $1B revenue."  ← one topic: history
Chunk 2: "The CEO joined in 2015... restructured the team..."      ← new topic: leadership
```
- ✅ Most meaningful chunks
- ❌ Slower, more complex to implement

**4. Recursive Chunking**
Try to split by paragraphs first → then sentences → then words, until chunks are small enough. Used by LangChain's default splitter.
```
Try split by "\n\n" (paragraphs) → too big?
  Try split by "\n" (lines) → still too big?
    Try split by ". " (sentences) → 
      Now within size limit ✅
```
- ✅ Respects natural document structure
- ✅ Most widely used in practice

---

## 6. Basic Retrieval + Generation Flow

Now let's put it all together. Here's the **complete RAG pipeline**:

```
┌─────────────────────────────────────────────────────┐
│                  INDEXING (done once)                │
│                                                      │
│  Documents → Chunk → Embed → Store in Vector DB      │
└─────────────────────────────────────────────────────┘

                          ↕  (at query time)

┌─────────────────────────────────────────────────────┐
│                  RETRIEVAL + GENERATION              │
│                                                      │
│  User Query                                          │
│      ↓                                               │
│  Embed the query → query vector                      │
│      ↓                                               │
│  Search Vector DB → Top K most similar chunks        │
│      ↓                                               │
│  Inject chunks into prompt:                          │
│  "Answer using only the context below:               │
│   [Chunk 1] [Chunk 2] [Chunk 3]                      │
│   Question: {user_query}"                            │
│      ↓                                               │
│  LLM generates answer grounded in retrieved chunks   │
│      ↓                                               │
│  Return answer (+ optionally cite source chunks)     │
└─────────────────────────────────────────────────────┘
```

### Concrete Example:

```
Documents indexed: Your company's 200-page HR policy PDF

User: "How many sick leaves do I get per year?"

Step 1: Embed query → vector
Step 2: Search HR policy chunks → finds:
        "Employees are entitled to 12 paid sick leaves 
         per calendar year, as per Section 5.3..."
Step 3: Prompt = "Answer using context: [above chunk]. 
                  Question: How many sick leaves per year?"
Step 4: LLM → "You get 12 paid sick leaves per year."
                ✅ Accurate. Grounded. Citable.
```

---

## How RAG Connects to Everything Before

```
Session 1: LLM predicts next token based on input
                    ↓
Session 2: Prompt Engineering shapes that input
                    ↓
Session 3: Agents give LLMs tools to act
                    ↓
Session 4: RAG is one of the most important tools
           an agent can have — the ability to 
           look things up from a knowledge base
           before answering.

RAG = the LLM's long-term memory for facts.
```

---

## Quick Recap — Session 4

| Concept | One-liner |
|---|---|
| RAG | Retrieve relevant docs → augment prompt → generate answer |
| Why RAG exists | LLMs have no live data, no private data, hallucinate |
| RAG vs Fine-tuning | RAG for facts & freshness; fine-tune for behaviour & style |
| Vector DB | Database that searches by meaning, not keywords |
| Embeddings | Text → numerical vectors capturing semantic meaning |
| Chunking | Split docs into pieces before embedding |
| Fixed chunking | Split every N tokens (simple but crude) |
| Sentence chunking | Split at sentence boundaries |
| Semantic chunking | Group by topic similarity |
| Recursive chunking | Try paragraph → sentence → word (most common) |
| RAG pipeline | Chunk → Embed → Index → Query → Retrieve → Generate |

---

Session 5 goes deeper — **Advanced Retrieval Strategies**, where you learn why basic RAG often fails in production and how to fix it. Things like query rewriting, re-ranking, HyDE, and corrective RAG.

Ready to continue?