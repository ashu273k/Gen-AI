# GenAI — Session 13: Deployment, Scaling & Final Demos

---

## 1. The Gap Between a Demo and Production

Every session so far has been about **building**. This session is about **shipping**.

```
Demo (what you've built so far):
  ✅ Runs on your laptop
  ✅ Works for 1 user at a time
  ✅ No authentication
  ✅ No cost controls
  ✅ Crashes gracefully — just restart it
  ✅ Latency doesn't matter much

Production (what you need to ship):
  ✅ Runs 24/7 without intervention
  ✅ Handles 10,000 users simultaneously  
  ✅ Authenticated and authorised
  ✅ Cost controlled and monitored
  ✅ Failures handled automatically
  ✅ Latency matters — users leave at > 3s
```

> The code that powers your capstone demo and the infrastructure that serves it to real users are two completely different engineering problems.

---

## 2. AWS Deployment Patterns for GenAI

AWS is the dominant cloud for production GenAI systems. Let's understand the key services and how they map to what you've built.

### The Core AWS Services You'll Use:

```
┌─────────────────────────────────────────────────────────┐
│              AWS GENAI DEPLOYMENT STACK                  │
│                                                          │
│  ┌──────────────┐  Entry point for all traffic         │
│  │ API Gateway  │  Rate limiting, auth, routing        │
│  └──────┬───────┘                                       │
│         ↓                                               │
│  ┌──────────────┐  Your agent application code         │
│  │   Lambda /   │  Serverless (Lambda) or              │
│  │     ECS      │  Containerised (ECS/Fargate)         │
│  └──────┬───────┘                                       │
│         ↓                                               │
│  ┌──────────────┐  LLM API calls                       │
│  │  Bedrock /   │  AWS Bedrock (managed LLMs) or       │
│  │External APIs │  Anthropic/OpenAI direct             │
│  └──────┬───────┘                                       │
│         ↓                                               │
│  ┌──────────────┐  Vector DB for RAG                   │
│  │  OpenSearch  │  (or Pinecone, managed externally)   │
│  │  / Pinecone  │                                       │
│  └──────┬───────┘                                       │
│         ↓                                               │
│  ┌──────────────┐  Conversation threads & memory       │
│  │  DynamoDB /  │  DynamoDB (KV, threads)              │
│  │     RDS      │  RDS (relational episodes)           │
│  └──────┬───────┘                                       │
│         ↓                                               │
│  ┌──────────────┐  Caching layer                       │
│  │ ElastiCache  │  Redis for response cache            │
│  │   (Redis)    │  and session management              │
│  └──────────────┘                                       │
└─────────────────────────────────────────────────────────┘
```

---

### Option 1 — Serverless (AWS Lambda)

Best for: **bursty, unpredictable traffic** — startup products, hackathon projects scaling up

```python
# Your agent as a Lambda function
import json
from agents import Agent, Runner, Thread

agent = Agent(
    name="Production Agent",
    instructions="...",
    tools=[...]
)

def lambda_handler(event, context):
    """AWS Lambda entry point."""
    
    try:
        # Parse request
        body = json.loads(event['body'])
        user_message = body['message']
        thread_id = body.get('thread_id')
        user_id = event['requestContext']['authorizer']['user_id']
        
        # Load thread from DynamoDB
        thread = load_thread_from_db(thread_id, user_id)
        
        # Run agent
        result = Runner.run_sync(
            agent,
            user_message,
            thread=thread
        )
        
        # Save updated thread
        save_thread_to_db(thread, user_id)
        
        return {
            'statusCode': 200,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'response': result.final_output,
                'thread_id': thread.thread_id
            })
        }
    
    except Exception as e:
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }
```

```
Lambda characteristics:
  ✅ Zero infrastructure management
  ✅ Auto-scales to millions of requests
  ✅ Pay only when running (not idle)
  ✅ Handles traffic spikes automatically
  ❌ Cold start latency (~500ms first request)
  ❌ 15 minute max execution time
  ❌ Not ideal for long-running agent workflows
```

---

### Option 2 — Containerised (ECS + Fargate)

Best for: **sustained traffic, long-running agents, complex dependencies**

```dockerfile
# Dockerfile for your agent service
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install -r requirements.txt

# Copy application code
COPY . .

# Health check endpoint
HEALTHCHECK --interval=30s --timeout=10s \
  CMD curl -f http://localhost:8000/health || exit 1

# Start FastAPI server
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```python
# main.py — FastAPI server wrapping your agent
from fastapi import FastAPI, Depends, HTTPException
from fastapi.responses import StreamingResponse
from pydantic import BaseModel

app = FastAPI()

class ChatRequest(BaseModel):
    message: str
    thread_id: str | None = None

@app.post("/chat")
async def chat(
    request: ChatRequest,
    user_id: str = Depends(verify_jwt_token)
):
    """Main agent endpoint with streaming support."""
    
    thread = await load_thread(request.thread_id, user_id)
    
    # Stream response back to client
    async def generate():
        async for token in Runner.run_stream(agent, request.message, thread=thread):
            yield f"data: {json.dumps({'token': token})}\n\n"
        yield "data: [DONE]\n\n"
    
    return StreamingResponse(generate(), media_type="text/event-stream")

@app.get("/health")
async def health():
    return {"status": "healthy"}
```

```
ECS + Fargate characteristics:
  ✅ No cold start latency
  ✅ Long-running processes supported
  ✅ Full control over runtime environment
  ✅ Easy horizontal scaling
  ✅ Streaming responses work naturally
  ❌ Slightly more complex to configure
  ❌ Pay even when idle (but use auto-scaling to minimise)
```

---

## 3. Scaling GenAI Services

Scaling AI services is different from scaling regular web services because of **LLM API costs and rate limits**.

### The Three Scaling Dimensions:

```
Dimension 1 — HORIZONTAL SCALING (more instances)
  Problem:  1 server handles 100 requests
  Solution: Run 10 servers, handle 1000 requests
  
  AWS tools: ECS Auto Scaling, ALB (Application Load Balancer)
  
  Key metric: CPU/memory per instance
  Trigger: Scale up when CPU > 70%

Dimension 2 — VERTICAL SCALING (bigger instances)
  Problem:  Agent needs more memory for large contexts
  Solution: Use larger instance type (more RAM, CPU)
  
  When to use: Single-threaded workloads, memory-bound

Dimension 3 — LLM-SPECIFIC SCALING (token throughput)
  Problem:  LLM API has rate limits (tokens per minute)
  Solution: Request queuing, multiple API keys, model routing
  
  This is unique to GenAI — doesn't exist in regular scaling
```

### LLM Rate Limit Management:

```python
import asyncio
from asyncio import Semaphore

class LLMRateLimiter:
    """
    Manage LLM API rate limits in production.
    Prevents hitting Anthropic/OpenAI rate limits.
    """
    
    def __init__(
        self,
        requests_per_minute: int = 60,
        tokens_per_minute: int = 100_000
    ):
        self.rpm_semaphore = Semaphore(requests_per_minute)
        self.token_budget = tokens_per_minute
        self.token_reset_interval = 60  # seconds
        
    async def acquire(self, estimated_tokens: int):
        """Acquire permission to make an LLM call."""
        async with self.rpm_semaphore:
            # Check token budget
            while self.token_budget < estimated_tokens:
                await asyncio.sleep(1)  # wait for budget to refresh
            
            self.token_budget -= estimated_tokens
            yield  # allow the API call

# Usage
rate_limiter = LLMRateLimiter(
    requests_per_minute=60,
    tokens_per_minute=100_000
)

async def run_agent_with_rate_limit(user_input: str):
    async with rate_limiter.acquire(estimated_tokens=2000):
        result = await Runner.run_async(agent, user_input)
    return result
```

### Multi-Model Load Balancing:

```python
class ModelRouter:
    """
    Route requests to different models based on
    complexity, cost, and availability.
    """
    
    def select_model(self, request: ChatRequest) -> str:
        
        # Simple queries → cheaper, faster model
        if self.is_simple_query(request.message):
            return "claude-haiku-4-5"    # cheapest, fastest
        
        # Complex reasoning → powerful model
        elif self.needs_deep_reasoning(request.message):
            return "claude-opus-4-6"     # most powerful
        
        # Default → balanced model
        else:
            return "claude-sonnet-4-6"   # balanced
    
    def is_simple_query(self, message: str) -> bool:
        simple_patterns = [
            len(message) < 50,
            "what is" in message.lower(),
            "define" in message.lower()
        ]
        return any(simple_patterns)

# This alone can cut LLM costs by 40-60%
router = ModelRouter()
```

---

## 4. Caching Strategies for GenAI

Caching is **dramatically more impactful** in GenAI than in regular web apps because LLM calls are expensive ($0.001–$0.05 per call).

### Three Layers of Caching:

```
Layer 1 — EXACT MATCH CACHE (Redis)
  Cache: Identical queries → identical responses
  Hit rate: ~5-15%
  Use for: FAQs, common greetings, static info queries
  
Layer 2 — SEMANTIC CACHE (Vector DB)
  Cache: Similar queries → same response
  Hit rate: ~20-40%
  Use for: Paraphrased versions of common questions
  
Layer 3 — PROMPT CACHE (Anthropic/OpenAI API)
  Cache: Repeated system prompts → reduced token cost
  Savings: Up to 90% on system prompt tokens
  Use for: Long system prompts sent with every request
```

### Semantic Cache Implementation:

```python
import chromadb
from datetime import datetime, timedelta

class SemanticCache:
    """
    Cache LLM responses by semantic similarity.
    "What's the weather?" and "How's the weather today?"
    return the same cached response.
    """
    
    def __init__(self, similarity_threshold: float = 0.95):
        self.collection = chromadb.Client().create_collection("response_cache")
        self.threshold = similarity_threshold
        self.ttl_hours = 24  # cache expires after 24 hours
    
    async def get(self, query: str) -> str | None:
        """Check if a semantically similar query was cached."""
        query_embedding = await embed(query)
        
        results = self.collection.query(
            query_embeddings=[query_embedding],
            n_results=1
        )
        
        if not results['distances'][0]:
            return None
        
        similarity = 1 - results['distances'][0][0]
        
        if similarity >= self.threshold:
            cached_data = json.loads(results['metadatas'][0][0]['data'])
            
            # Check TTL
            if datetime.fromisoformat(cached_data['cached_at']) > \
               datetime.now() - timedelta(hours=self.ttl_hours):
                return cached_data['response']
        
        return None
    
    async def set(self, query: str, response: str):
        """Cache a query-response pair."""
        query_embedding = await embed(query)
        
        self.collection.add(
            embeddings=[query_embedding],
            documents=[query],
            metadatas=[{
                'data': json.dumps({
                    'response': response,
                    'cached_at': datetime.now().isoformat()
                })
            }],
            ids=[f"cache_{hash(query)}"]
        )

# Usage in agent endpoint
cache = SemanticCache(similarity_threshold=0.95)

async def cached_agent_run(user_input: str) -> str:
    
    # Check cache first
    cached_response = await cache.get(user_input)
    if cached_response:
        return cached_response  # ~5ms vs ~1500ms
    
    # Cache miss — run agent
    result = await Runner.run_async(agent, user_input)
    response = result.final_output
    
    # Store in cache
    await cache.set(user_input, response)
    
    return response
```

### Anthropic Prompt Caching:

```python
# Prefix caching — Anthropic caches repeated prompt prefixes
# Your long system prompt only gets processed once per hour

response = anthropic.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": very_long_system_prompt,  # 10,000 tokens
            "cache_control": {"type": "ephemeral"}  # ← tell Anthropic to cache this
        }
    ],
    messages=[{"role": "user", "content": user_message}]
)

# First call: charges full 10,000 tokens for system prompt
# Subsequent calls within 1 hour: 90% discount on system prompt tokens
# Savings at scale: significant for long system prompts
```

---

## 5. Rate Limiting

Rate limiting protects your system from abuse and runaway costs.

### Three Levels of Rate Limiting:

```
Level 1 — API Gateway Rate Limiting
  → Maximum requests per second per IP
  → Blocks DDoS before reaching your code
  → AWS API Gateway: set in console, costs nothing

Level 2 — Per-User Rate Limiting
  → Maximum LLM calls per user per day
  → Prevents single user exhausting your LLM budget
  → Stored in Redis

Level 3 — Cost Rate Limiting
  → Maximum spend per user per month
  → Hard stop when user hits their plan limit
  → Stored in database
```

```python
import redis
from datetime import datetime

redis_client = redis.Redis()

class RateLimiter:
    
    def check_user_rate_limit(
        self,
        user_id: str,
        plan: str = "free"
    ) -> tuple[bool, dict]:
        """
        Check if user has exceeded their rate limits.
        Returns (allowed: bool, limit_info: dict)
        """
        
        limits = {
            "free":       {"daily_calls": 20,  "monthly_tokens": 100_000},
            "pro":        {"daily_calls": 500, "monthly_tokens": 5_000_000},
            "enterprise": {"daily_calls": -1,  "monthly_tokens": -1}  # unlimited
        }
        
        user_limits = limits[plan]
        today = datetime.now().strftime("%Y-%m-%d")
        month = datetime.now().strftime("%Y-%m")
        
        # Check daily call limit
        daily_key = f"rate:calls:{user_id}:{today}"
        daily_calls = int(redis_client.get(daily_key) or 0)
        
        if user_limits["daily_calls"] != -1:
            if daily_calls >= user_limits["daily_calls"]:
                return False, {
                    "error": "daily_limit_exceeded",
                    "limit": user_limits["daily_calls"],
                    "used": daily_calls,
                    "resets_at": "midnight IST"
                }
        
        # Check monthly token limit
        monthly_key = f"rate:tokens:{user_id}:{month}"
        monthly_tokens = int(redis_client.get(monthly_key) or 0)
        
        if user_limits["monthly_tokens"] != -1:
            if monthly_tokens >= user_limits["monthly_tokens"]:
                return False, {
                    "error": "monthly_token_limit_exceeded",
                    "limit": user_limits["monthly_tokens"],
                    "used": monthly_tokens,
                    "resets_at": "1st of next month"
                }
        
        # Increment counters
        pipe = redis_client.pipeline()
        pipe.incr(daily_key)
        pipe.expire(daily_key, 86400)     # expire after 24 hours
        pipe.execute()
        
        return True, {"daily_remaining": user_limits["daily_calls"] - daily_calls - 1}

# Usage in endpoint
rate_limiter = RateLimiter()

@app.post("/chat")
async def chat(request: ChatRequest, user = Depends(get_current_user)):
    
    allowed, info = rate_limiter.check_user_rate_limit(
        user.id, user.plan
    )
    
    if not allowed:
        raise HTTPException(status_code=429, detail=info)
    
    result = await Runner.run_async(agent, request.message)
    return {"response": result.final_output}
```

---

## 6. CI/CD for AI Systems

CI/CD for AI systems has an extra layer beyond regular software — you need to test **model behaviour**, not just code correctness.

### The AI CI/CD Pipeline:

```
Code Push to GitHub
       ↓
┌──────────────────────────────────────────┐
│  STAGE 1 — CODE TESTS (standard)         │
│  → Unit tests for tools and utilities    │
│  → Integration tests for API endpoints  │
│  → Linting, type checking               │
│  Duration: ~2 minutes                   │
└──────────────────────┬───────────────────┘
                       ↓
┌──────────────────────────────────────────┐
│  STAGE 2 — PROMPT REGRESSION TESTS      │
│  → Run 50 golden test cases             │
│  → Compare outputs to expected results  │
│  → Flag any degraded responses          │
│  Duration: ~5 minutes                   │
└──────────────────────┬───────────────────┘
                       ↓
┌──────────────────────────────────────────┐
│  STAGE 3 — LLM EVALUATION               │
│  → LLM judge scores response quality    │
│  → Check hallucination rate             │
│  → Check tool call accuracy             │
│  → Must score > 85% to proceed          │
│  Duration: ~10 minutes                  │
└──────────────────────┬───────────────────┘
                       ↓
┌──────────────────────────────────────────┐
│  STAGE 4 — COST ESTIMATION              │
│  → Estimate tokens per average request  │
│  → Project monthly cost at scale        │
│  → Alert if cost increased > 20%        │
│  Duration: ~1 minute                    │
└──────────────────────┬───────────────────┘
                       ↓
┌──────────────────────────────────────────┐
│  STAGE 5 — DEPLOY                       │
│  → Blue/green deployment                │
│  → 5% traffic to new version first      │
│  → Monitor error rate for 10 minutes    │
│  → If stable → 100% traffic cutover     │
│  → If errors → automatic rollback       │
│  Duration: ~15 minutes                  │
└──────────────────────────────────────────┘
```

### Golden Test Suite:

```python
# eval/golden_tests.py — your regression test suite

GOLDEN_TESTS = [
    {
        "id": "gt_001",
        "input": "What is the leave policy?",
        "expected_tools": ["search_hr_docs"],
        "expected_contains": ["annual leave", "sick leave"],
        "expected_not_contains": ["I don't know", "I cannot"],
        "max_response_time_ms": 3000
    },
    {
        "id": "gt_002", 
        "input": "Send an email to Priya about tomorrow's meeting",
        "expected_tools": ["send_email"],
        "expected_tool_args": {
            "send_email": {"to": lambda x: "priya" in x[0].lower()}
        },
        "max_response_time_ms": 5000
    },
    {
        "id": "gt_003",
        "input": "What's 2 + 2?",  # simple query
        "expected_tools": [],       # should NOT call any tools
        "expected_contains": ["4"],
        "max_response_time_ms": 2000
    }
]

async def run_regression_tests(agent) -> dict:
    results = {"passed": 0, "failed": 0, "failures": []}
    
    for test in GOLDEN_TESTS:
        start = time.time()
        result = await Runner.run_async(agent, test["input"])
        elapsed_ms = (time.time() - start) * 1000
        
        # Check response time
        if elapsed_ms > test["max_response_time_ms"]:
            results["failed"] += 1
            results["failures"].append(
                f"{test['id']}: Too slow ({elapsed_ms:.0f}ms)"
            )
            continue
        
        # Check expected tools were called
        tools_called = [tc.name for tc in result.tool_calls]
        for expected_tool in test.get("expected_tools", []):
            if expected_tool not in tools_called:
                results["failed"] += 1
                results["failures"].append(
                    f"{test['id']}: Tool {expected_tool} not called"
                )
                continue
        
        # Check response content
        response = result.final_output.lower()
        for phrase in test.get("expected_contains", []):
            if phrase.lower() not in response:
                results["failed"] += 1
                results["failures"].append(
                    f"{test['id']}: Missing '{phrase}' in response"
                )
                break
        else:
            results["passed"] += 1
    
    return results
```

---

## 7. Production Monitoring Dashboard

Once deployed, you need visibility into your system's health:

```
PRODUCTION MONITORING METRICS:

┌─────────────────────────────────────────────────────────┐
│  REAL-TIME DASHBOARD                                     │
│                                                          │
│  TRAFFIC              LATENCY              COST          │
│  Req/min: 847         P50:  890ms          $/hr: $2.34  │
│  Active:  234         P95:  2100ms         $/day: $56.1 │
│  Errors:  0.3%        P99:  4200ms         $/mo est:$1.7k│
│                                                          │
│  AGENT QUALITY        TOOL HEALTH          CACHE         │
│  Quality score: 91%   Success rate: 98.2%  Hit rate: 34%│
│  Hallucination: 1.2%  Avg latency: 340ms   Saves: $18/d │
│  Refusals: 0.8%       Failures: gmail(2)   Size: 12k     │
│                                                          │
│  ALERTS                                                  │
│  🟡 P99 latency elevated (4.2s > 3s threshold)          │
│  🟡 Gmail tool failure rate 1.8% (threshold: 1%)        │
│  🟢 All other metrics normal                             │
└─────────────────────────────────────────────────────────┘
```

---

## 8. The Complete Production Architecture

Everything from all 13 sessions, deployed:

```
┌─────────────────────────────────────────────────────────────┐
│                COMPLETE PRODUCTION SYSTEM                    │
│                                                              │
│  USER                                                        │
│    ↓                                                         │
│  CloudFront (CDN) → serves frontend assets                   │
│    ↓                                                         │
│  API Gateway → rate limiting, auth, SSL termination          │
│    ↓                                                         │
│  Application Load Balancer → distributes across instances    │
│    ↓                                                         │
│  ECS Fargate Cluster (your agent service)                    │
│    ├── Instance 1  ─┐                                        │
│    ├── Instance 2   ├── Auto-scales 2-20 instances           │
│    └── Instance N  ─┘                                        │
│    ↓                                                         │
│  ┌─────────────────────────────────────────────┐            │
│  │  AGENT LAYER                                 │            │
│  │  Agent SDK + MCP servers                     │            │
│  │  Session 7 + Session 12                      │            │
│  └─────────────────────────────────────────────┘            │
│    ↓                    ↓                   ↓               │
│  Anthropic API     ElastiCache          DynamoDB             │
│  (LLM calls)       (Redis cache)        (threads/memory)     │
│  Sessions 1-2      Session 13           Sessions 8-9         │
│    ↓                                        ↓               │
│  OpenSearch/           Neo4j            RDS PostgreSQL        │
│  Pinecone              (graph DB)       (episode history)     │
│  (vector store)        Session 10       Session 9            │
│  Sessions 4-5                                                │
│    ↓                                                         │
│  CloudWatch + Langfuse (observability)                       │
│  Session 8                                                   │
└─────────────────────────────────────────────────────────────┘
```

---

## The Complete GenAI Course — Everything Connected

```
LAYER 1 — FOUNDATION
  Session 1: How LLMs work internally
  Session 2: How to steer them with prompts

LAYER 2 — AGENTS
  Session 3: Agents — LLMs that can act
  Session 6: LangChain/LangGraph — orchestration
  Session 7: Agent SDK — production agents

LAYER 3 — KNOWLEDGE
  Session 4: RAG — giving agents documents
  Session 5: Advanced RAG — production retrieval
  Session 10: Graph DBs — relationship knowledge

LAYER 4 — MEMORY
  Session 8: Threads & Tracing — session memory
  Session 9: Memory Systems — cross-session memory

LAYER 5 — INTERFACES
  Session 11: Voice Agents — audio I/O
  Session 12: MCP — standard tool protocol

LAYER 6 — PRODUCTION
  Session 13: Deployment, Scaling, CI/CD, Monitoring
              (this session)
```

---

## Quick Recap — Session 13

| Concept | One-liner |
|---|---|
| Demo vs production | Demo works for 1 user; production works for 10,000 concurrently |
| Lambda | Serverless, auto-scales, cold start latency, 15min limit |
| ECS + Fargate | Containerised, no cold start, streaming-friendly, always on |
| Horizontal scaling | More instances — handled by ECS Auto Scaling + ALB |
| LLM rate limiting | Manage tokens-per-minute limits with semaphores and queuing |
| Model routing | Route simple queries to cheap models, complex to powerful ones |
| Exact match cache | Redis cache for identical queries — ~5ms vs ~1500ms |
| Semantic cache | Vector DB cache for similar queries — 20-40% hit rate |
| Prompt caching | Anthropic caches repeated system prompts — 90% token discount |
| Per-user rate limiting | Redis counters per user per day/month — prevents abuse |
| CI/CD for AI | Code tests + prompt regression + LLM evaluation + cost check + deploy |
| Golden tests | Fixed input/output pairs that must pass before every deployment |
| Blue/green deploy | 5% traffic to new version first, monitor, then full cutover |
| Production monitoring | Track latency, cost, quality score, tool health, cache hit rate |

---

## 🎓 Course Complete — GenAI Sessions 1–13

You've now covered the **complete GenAI engineering curriculum**:

```
From understanding how a transformer predicts one token
                      ↓
To deploying a multi-agent, voice-enabled, RAG-powered,
graph-memory-backed, MCP-connected production system
on AWS with CI/CD, monitoring, caching, and rate limiting.

That is the full stack of modern GenAI engineering.
```

Your remaining sessions in the syllabus cover:
- **System Design (15 sessions)** — HLD, databases, distributed systems
- **Deep Learning (16 sessions)** — neural networks, CNNs, transformers, BERT/GPT

Which subject would you like to tackle next?