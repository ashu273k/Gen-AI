# GenAI — Session 10: Graph Databases & Relational Memory

---

## 1. The Problem With What We Have So Far

Before introducing graph databases, let's understand exactly what they solve that previous memory strategies cannot.

Recall from Session 9 — our memory strategies:
```
Key-Value Store → stores FACTS
  "Rahul works at a fintech startup"
  "Rahul is building a fraud detection system"

Vector Store → stores EXPERIENCES
  "Past conversation about RAG"
  "Past conversation about deployment"

Database → stores EPISODES
  "Session on 1st June covered chunking strategies"
```

Now ask a slightly complex question:

```
"Which engineers in our team have worked on projects 
 that use the same tech stack as Rahul's fraud system?"
```

With key-value or vector stores:
```
❌ You'd need to:
   → Fetch Rahul's tech stack
   → Fetch every engineer's tech stack
   → Manually compare
   → Filter for matches

This is a RELATIONSHIP query.
None of our storage strategies model relationships.
They store isolated facts, not connected knowledge.
```

> **The core limitation:** Traditional databases store facts. Graph databases store **facts AND their relationships** — and make querying relationships as fast as querying facts.

---

## 2. Graph DB Fundamentals

A graph database stores data as a **network of nodes and edges** — not rows and columns.

### The Three Core Concepts:

**Nodes** — entities (things that exist)
```
(Rahul) (TCS) (Python) (FraudSystem) (RAG) (Priya)
```

**Edges** — relationships between entities (directed, named)
```
(Rahul) -[WORKS_AT]→ (TCS)
(Rahul) -[BUILDS]→ (FraudSystem)
(FraudSystem) -[USES_TECH]→ (Python)
(FraudSystem) -[USES_TECH]→ (RAG)
(Priya) -[WORKS_AT]→ (TCS)
(Priya) -[BUILDS]→ (PaymentSystem)
(PaymentSystem) -[USES_TECH]→ (Python)
```

**Properties** — attributes on nodes and edges
```
(Rahul {name: "Rahul Sharma", role: "Senior PM", 
        expertise: "intermediate"})
        
-[WORKS_AT {since: "2023", department: "Engineering"}]→

(TCS {industry: "IT Services", size: "large"})
```

### Visualised:

```
        [Python]
           ↑           ↑
    USES_TECH       USES_TECH
           |               |
[Rahul]→BUILDS→[FraudSystem]   [PaymentSystem]←BUILDS←[Priya]
   |                                                      |
WORKS_AT                                              WORKS_AT
   ↓                                                      ↓
  [TCS] ←──────────────── same company ──────────────── [TCS]
```

Now that relationship question becomes trivial:
```
"Find engineers who work at TCS and build systems 
 that use Python like Rahul's FraudSystem"

→ Follow edges:
  Rahul → BUILDS → FraudSystem → USES_TECH → Python
  ← USES_TECH ← PaymentSystem ← BUILDS ← Priya
  Priya → WORKS_AT → TCS ✓
  
Answer: Priya ✅  (2 hops through the graph)
```

In a relational database, this would require multiple JOIN operations. In a graph DB, it's a natural traversal.

---

## 3. Knowledge Graphs for AI

A **knowledge graph** is a specific type of graph database designed to store **structured world knowledge** that AI systems can reason over.

Think of it as the AI equivalent of a human's mental model of the world:

```
Human mental model:
  "Infosys is an IT company"
  "IT companies hire software engineers"  
  "Software engineers use Python"
  "Python is used for AI/ML"
  → Can reason: "Infosys probably has AI/ML work"

Knowledge graph:
  (Infosys)-[IS_A]→(IT_Company)
  (IT_Company)-[HIRES]→(SoftwareEngineer)
  (SoftwareEngineer)-[USES]→(Python)
  (Python)-[USED_FOR]→(AI_ML)
  → Agent can traverse this graph to reach the same conclusion
```

### Why Knowledge Graphs Matter for Agents

```
Without knowledge graph:
  Agent knows isolated facts
  Can't infer relationships between facts
  Each query starts from scratch
  
With knowledge graph:
  Agent knows facts AND their connections
  Can traverse relationships to infer new facts
  Can answer multi-hop questions naturally
  Context is rich and interconnected
```

### Real Knowledge Graph Example — Medical AI:

```
(Metformin)-[TREATS]→(Type2Diabetes)
(Metformin)-[CONTRAINDICATED_WITH]→(KidneyDisease)
(Rahul)-[HAS_CONDITION]→(Type2Diabetes)
(Rahul)-[HAS_CONDITION]→(ChronicKidneyDisease)

Agent query: "Is Metformin safe for Rahul?"

Graph traversal:
  Metformin → TREATS → Type2Diabetes ✓ (Rahul has this)
  Metformin → CONTRAINDICATED_WITH → KidneyDisease ✗ (Rahul has this too!)
  
Agent: "Metformin treats Rahul's diabetes but is contraindicated 
        with his kidney disease. Consult a doctor before prescribing." ✅

No vector search could reliably surface this multi-hop reasoning.
```

---

## 4. Relational vs Graph Memory

Let's make this concrete with a direct comparison.

### Scenario: AI agent for a software engineering team

**Storing with Relational DB (PostgreSQL):**

```sql
-- Three separate tables
CREATE TABLE engineers (
    id INT, name VARCHAR, role VARCHAR, company VARCHAR
);

CREATE TABLE projects (
    id INT, name VARCHAR, engineer_id INT, status VARCHAR
);

CREATE TABLE technologies (
    id INT, name VARCHAR, project_id INT
);

-- To answer: "Who works on Python projects at our company?"
SELECT e.name 
FROM engineers e
JOIN projects p ON e.id = p.engineer_id
JOIN technologies t ON p.id = t.project_id
WHERE t.name = 'Python' 
AND e.company = 'our_company';

-- 2 JOINs for a simple question
-- 5 hops of relationships → exponentially more JOINs
-- Performance degrades rapidly with depth
```

**Storing with Graph DB (Neo4j):**

```cypher
-- The same question in Cypher (Neo4j's query language)
MATCH (e:Engineer)-[:WORKS_AT]→(c:Company {name: "our_company"}),
      (e)-[:BUILDS]→(p:Project)-[:USES_TECH]→(t:Tech {name: "Python"})
RETURN e.name

-- Reads like English: 
-- "Find engineers who work at our company 
--  and build projects that use Python"
-- Performance stays constant regardless of depth
```

### The Performance Difference:

```
Query depth    Relational DB    Graph DB
─────────────────────────────────────────
2 hops         Fast             Fast
3 hops         Moderate         Fast
4 hops         Slow             Fast
5 hops         Very slow        Fast
10 hops        Timeout          Fast

Graph DBs are optimised for TRAVERSAL.
Relational DBs are optimised for AGGREGATION.
```

### When to Use Which:

```
Use Relational DB when:
  ✅ Data is highly structured with known schema
  ✅ Queries are mostly aggregations (SUM, COUNT, AVG)
  ✅ Relationships are simple and shallow (1-2 hops)
  ✅ ACID transactions are critical
  ✅ Reporting and analytics workloads

Use Graph DB when:
  ✅ Relationships are as important as the data itself
  ✅ Queries traverse many hops (3+)
  ✅ Schema evolves — new relationship types emerge
  ✅ You're modelling a network (social, knowledge, dependency)
  ✅ Recommendation systems
  ✅ Fraud detection (finding suspicious connection patterns)
```

---

## 5. Neo4j — Deep Dive

**Neo4j** is the most widely used graph database, and the one your syllabus specifically covers. Let's understand it properly.

### Neo4j Architecture:

```
┌─────────────────────────────────────────────┐
│               NEO4J ARCHITECTURE            │
│                                             │
│  ┌─────────────┐    ┌─────────────────┐    │
│  │   Cypher    │    │   Graph Engine  │    │
│  │Query Language│   │  (Native Graph  │    │
│  └──────┬──────┘    │    Storage)     │    │
│         │           └────────┬────────┘    │
│         ↓                    │             │
│  ┌─────────────────────────────────────┐   │
│  │         Query Planner               │   │
│  │  (optimises traversal paths)        │   │
│  └──────────────────┬──────────────────┘   │
│                     ↓                      │
│  ┌─────────────────────────────────────┐   │
│  │      Native Graph Storage           │   │
│  │  Nodes and edges stored as          │   │
│  │  PHYSICAL POINTERS to each other    │   │
│  │  (not as rows in a table)           │   │
│  └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

The key insight: Neo4j stores **physical pointers** between nodes. Traversing a relationship means following a pointer — O(1) operation — not scanning a table.

---

### Cypher Query Language

Cypher is Neo4j's query language. It's designed to be **visual** — queries look like the graph they describe.

**Creating nodes and relationships:**
```cypher
// Create nodes
CREATE (rahul:Person {name: "Rahul", role: "PM"})
CREATE (fraud:Project {name: "FraudSystem", status: "active"})
CREATE (python:Technology {name: "Python"})
CREATE (rag:Technology {name: "RAG"})

// Create relationships
CREATE (rahul)-[:BUILDS {since: "2025-01"}]->(fraud)
CREATE (fraud)-[:USES_TECH]->(python)
CREATE (fraud)-[:USES_TECH]->(rag)
```

**Querying — simple:**
```cypher
// Find all projects Rahul is building
MATCH (p:Person {name: "Rahul"})-[:BUILDS]->(proj:Project)
RETURN proj.name, proj.status
```

**Querying — multi-hop:**
```cypher
// Find all technologies used by projects 
// built by people who work at the same company as Rahul
MATCH (rahul:Person {name: "Rahul"})-[:WORKS_AT]->(company)
      <-[:WORKS_AT]-(colleague:Person)
      -[:BUILDS]->(proj:Project)
      -[:USES_TECH]->(tech:Technology)
RETURN colleague.name, proj.name, tech.name

// This would be 4 JOINs in SQL
// In Cypher: one readable MATCH clause
```

**Pattern matching — fraud detection example:**
```cypher
// Find suspicious transaction patterns:
// accounts that received money from flagged accounts
// within 24 hours
MATCH (flagged:Account {status: "flagged"})
      -[:SENT {timestamp: $within_24h}]->(intermediate)
      -[:SENT]->(suspect:Account)
WHERE suspect.status <> "flagged"
RETURN suspect.id, suspect.name, 
       COUNT(*) as suspicious_transactions
ORDER BY suspicious_transactions DESC
```

---

## 6. Neo4j Use Cases in Agents

### Use Case 1 — Agent Personal Knowledge Graph

Build a personal knowledge graph that grows as the agent learns about the user:

```python
from neo4j import GraphDatabase

class AgentMemoryGraph:
    
    def __init__(self):
        self.driver = GraphDatabase.driver(
            "bolt://localhost:7687",
            auth=("neo4j", "password")
        )
    
    def add_person(self, name: str, role: str):
        with self.driver.session() as session:
            session.run("""
                MERGE (p:Person {name: $name})
                SET p.role = $role
            """, name=name, role=role)
    
    def add_works_on(self, person: str, project: str, tech_stack: list):
        with self.driver.session() as session:
            # Create project and connect to person
            session.run("""
                MATCH (p:Person {name: $person})
                MERGE (proj:Project {name: $project})
                MERGE (p)-[:WORKS_ON]->(proj)
            """, person=person, project=project)
            
            # Connect technologies
            for tech in tech_stack:
                session.run("""
                    MATCH (proj:Project {name: $project})
                    MERGE (t:Technology {name: $tech})
                    MERGE (proj)-[:USES]->(t)
                """, project=project, tech=tech)
    
    def find_colleagues_with_shared_tech(self, person: str):
        with self.driver.session() as session:
            result = session.run("""
                MATCH (p:Person {name: $person})-[:WORKS_ON]
                      ->(proj)-[:USES]->(tech)
                      <-[:USES]-(other_proj)
                      <-[:WORKS_ON]-(colleague:Person)
                WHERE colleague.name <> $person
                RETURN colleague.name, 
                       collect(tech.name) as shared_tech
            """, person=person)
            return result.data()

# Agent uses this as a tool
memory_graph = AgentMemoryGraph()

@function_tool
def find_team_members_with_expertise(technology: str) -> str:
    """Find team members who have experience with a specific technology."""
    results = memory_graph.query_by_tech(technology)
    return str(results)
```

---

### Use Case 2 — Fraud Detection Agent

The syllabus specifically mentions fraud detection. Here's how a graph powers it:

```
Traditional approach (rule-based):
  IF transaction > ₹1,00,000 AND new_account → flag
  → Catches simple fraud
  → Misses sophisticated ring patterns

Graph approach:
  Detect: account A → sends to B → sends to C → sends to A
  (circular money flow = money laundering pattern)
  
  Detect: single IP address → controls 50 accounts
  (bot network pattern)
  
  Detect: new account → receives from 100 accounts in 1 hour
  (aggregation fraud pattern)
```

```cypher
-- Detect circular money flow (money laundering)
MATCH path = (a:Account)-[:SENT*3..6]->(a)
WHERE ALL(tx IN relationships(path) 
          WHERE tx.amount > 10000
          AND tx.timestamp > datetime() - duration('P1D'))
RETURN a.id, length(path) as cycle_length, 
       [tx IN relationships(path) | tx.amount] as amounts
```

```python
@function_tool
def detect_fraud_patterns(account_id: str) -> str:
    """
    Analyse transaction graph for fraud patterns including
    circular flows, velocity anomalies, and network clusters.
    """
    patterns = fraud_graph.find_patterns(account_id)
    if patterns:
        return f"⚠️ Suspicious patterns found: {patterns}"
    return "No suspicious patterns detected"
```

---

### Use Case 3 — Recommendation Engine

```cypher
-- "Users who liked similar items also liked..."
-- Classic collaborative filtering in graph form

MATCH (user:User {id: $user_id})-[:LIKED]->(item:Item)
      <-[:LIKED]-(similar_user:User)
      -[:LIKED]->(recommendation:Item)
WHERE NOT (user)-[:LIKED]->(recommendation)
AND NOT (user)-[:DISLIKED]->(recommendation)
RETURN recommendation.name, 
       COUNT(similar_user) as recommendation_strength
ORDER BY recommendation_strength DESC
LIMIT 10
```

---

### Use Case 4 — Code Dependency Agent

```cypher
-- Map code dependencies for an AI code review agent
CREATE (auth:Module {name: "auth_service"})
CREATE (db:Module {name: "database_service"})  
CREATE (api:Module {name: "api_gateway"})

CREATE (api)-[:DEPENDS_ON]->(auth)
CREATE (api)-[:DEPENDS_ON]->(db)
CREATE (auth)-[:DEPENDS_ON]->(db)

-- "What breaks if database_service changes?"
MATCH (db:Module {name: "database_service"})
      <-[:DEPENDS_ON*1..]-(affected:Module)
RETURN DISTINCT affected.name
-- Returns: auth_service, api_gateway
-- Agent knows what to retest when db changes
```

---

## 7. Connecting Graph Memory to the Full Agent Architecture

```
COMPLETE AGENT MEMORY ARCHITECTURE (updated):

┌─────────────────────────────────────────────────┐
│              MEMORY RETRIEVAL                    │
│                                                  │
│  ┌─────────────┐  Fast structured facts         │
│  │  Key-Value  │  (user profile, preferences)   │
│  │   Store     │                                 │
│  └─────────────┘                                 │
│                                                  │
│  ┌─────────────┐  Semantic similarity search     │
│  │   Vector    │  (past conversations, docs)     │
│  │   Store     │                                 │
│  └─────────────┘                                 │
│                                                  │
│  ┌─────────────┐  Episode history               │
│  │  Relational │  (full session records)         │
│  │     DB      │                                 │
│  └─────────────┘                                 │
│                                                  │
│  ┌─────────────┐  ← NEW                         │
│  │   Graph DB  │  Relationship traversal         │
│  │   (Neo4j)   │  (connected knowledge,          │
│  └─────────────┘   multi-hop reasoning)          │
│                                                  │
└──────────────────────┬──────────────────────────┘
                       ↓
              Context Window
                       ↓
                     Agent
```

Each memory layer answers a different question:
```
KV Store:    "Who is this user?"
Vector Store: "What have we talked about before?"
Relational:  "What happened in past sessions?"
Graph DB:    "How are things connected to each other?"
```

---

## Sessions 9 & 10 Connection

```
Session 9 — Memory Systems:
  HOW to store and retrieve agent memory
  Four strategies: KV, Vector, DB, In-context
  All store FACTS and EXPERIENCES

Session 10 — Graph Databases:
  Extends memory to store RELATIONSHIPS
  Not just "Rahul uses Python"
  But "Rahul uses Python, which connects him to 
       these 5 colleagues, this project, this domain"
       
  Graph memory = the agent's ability to REASON
  about how things in the world connect to each other
  
  Knowledge graph + Vector store = 
  the most powerful agent memory combination
```

---

## Quick Recap — Session 10

| Concept | One-liner |
|---|---|
| Graph DB motivation | Traditional stores facts; graph stores facts AND relationships |
| Node | An entity in the graph (person, project, technology) |
| Edge | A named, directed relationship between two nodes |
| Property | Attribute on a node or edge |
| Knowledge graph | Graph that stores structured world knowledge for AI reasoning |
| Relational vs Graph | Relational = aggregation; Graph = traversal |
| Performance | Graph traversal stays O(1) per hop regardless of data size |
| Neo4j | Most widely used graph DB — native graph storage with physical pointers |
| Cypher | Neo4j's query language — queries look like the graphs they describe |
| Multi-hop queries | Graph answers 5-hop questions trivially; SQL needs 4 JOINs |
| Fraud detection | Graph detects ring patterns, circular flows, network clusters |
| Agent memory stack | KV + Vector + Relational + Graph = complete memory architecture |

---

Session 11 moves to **Conversational & Voice-Based AI Agents** — speech-to-text pipelines, real-time voice conversations, streaming, and latency handling. Ready?