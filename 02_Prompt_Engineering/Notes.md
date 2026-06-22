# GenAI — Session 2: Prompt Engineering

---

## 1. The GIGO Principle

**GIGO = Garbage In, Garbage Out**

This is the foundational principle of prompt engineering.

> The model is only as good as the instructions you give it. A brilliant model with a terrible prompt gives a terrible output.

```
Weak prompt:  "Write something about climate"
Strong prompt: "Write a 3-paragraph explainer for a 16-year-old 
                on how rising CO2 levels cause ocean acidification, 
                using a simple analogy in the first paragraph."
```

Same model. Completely different output quality. **The prompt is the product.**

---

## 2. Prompt Styles

Different teams/companies standardise how they format prompts for their models. Think of these as "dialects":

### Alpaca Format
Developed by Stanford. Used for instruction-tuned models.
```
### Instruction:
Summarise the following text.

### Input:
[your text here]

### Response:
```

### INST Format
Used by Meta's LLaMA/Mistral models:
```
[INST] Summarise the following text: [your text here] [/INST]
```

### FLAN-T5 Format
Google's format — very minimal, task-prefix based:
```
summarize: [your text here]
```

### ⭐ ChatML — The Industry Standard
Used by OpenAI, and now widely adopted across the industry. This is what you'll actually use:
```
<|im_start|>system
You are a helpful assistant.
<|im_end|>
<|im_start|>user
Summarise this text for me.
<|im_end|>
<|im_start|>assistant
```

Every modern LLM API (Claude, GPT, Gemini) maps to this structure under the hood — a **system message**, followed by alternating **user** and **assistant** turns.

---

## 3. System vs User Prompts

This is one of the most important distinctions in applied LLM work:

| | System Prompt | User Prompt |
|---|---|---|
| **Who writes it** | Developer / You | End user |
| **Purpose** | Sets persona, rules, constraints, tone | The actual request/question |
| **Visibility** | Usually hidden from end user | Visible |
| **Persistence** | Stays constant across the conversation | Changes every turn |

```
System:  "You are a senior tax consultant in India. 
          Always cite the relevant section of the Income Tax Act.
          Never give advice outside Indian tax law."

User:    "Can I claim HRA if I'm working from home?"
```

> The system prompt is your **constitution** for the model. It defines the rules of the game before the user ever types anything.

---

## 4. Zero-Shot vs Few-Shot Prompting

### Zero-Shot
You give the model a task with **no examples**. You rely entirely on what the model learned during training.

```
Prompt: "Classify the sentiment of this review as 
         Positive, Negative, or Neutral:
         'The delivery was late but the product is great.'"

Output: "Mixed / Neutral"
```

### Few-Shot
You give the model **2–5 examples** of the task before asking it to do the real one. This "shows" the model the pattern you want.

```
Prompt:
Review: "Absolutely loved it!" → Positive
Review: "Waste of money."      → Negative
Review: "It's okay I guess."   → Neutral

Now classify:
Review: "The delivery was late but the product is great." → ?
```

**Why few-shot works:** The model picks up on the *format*, *reasoning style*, and *output structure* from your examples — without any retraining.

> Rule of thumb: Use zero-shot first. If outputs are inconsistent, switch to few-shot with 3–5 diverse examples.

---

## 5. Chain-of-Thought (CoT) Prompting

Standard prompting asks for the answer directly. CoT prompting asks the model to **think step by step** before answering.

### Without CoT:
```
Q: Roger has 5 tennis balls. He buys 2 more cans of 3 balls each. 
   How many does he have?
A: 11   ✓  (got lucky)
```

### With CoT:
```
Q: Roger has 5 tennis balls. He buys 2 more cans of 3 balls each. 
   How many does he have? Think step by step.

A: Roger starts with 5 balls.
   He buys 2 cans × 3 balls = 6 balls.
   5 + 6 = 11 balls.   ✓ (reliable)
```

For complex reasoning tasks, CoT dramatically improves accuracy because the model is forced to **decompose the problem** rather than jump to an answer.

The magic trigger phrase: **"Think step by step"** or **"Let's reason through this."**

---

## 6. Self-Consistency

A technique built on top of CoT.

**The idea:** Run the same CoT prompt **multiple times** (with slightly different sampling), get multiple reasoning paths, and take the **majority vote** answer.

```
Run 1: ... → Answer: 11
Run 2: ... → Answer: 11  
Run 3: ... → Answer: 9   ← outlier
Run 4: ... → Answer: 11
Run 5: ... → Answer: 11

Majority vote → 11 ✓
```

> Useful when accuracy matters more than speed. It's basically asking the model to "check its own work" via multiple independent attempts.

---

## 7. Persona-Based Prompting

You assign the model a **specific identity or role** to shift its tone, expertise level, and response style.

```
"You are a no-nonsense senior SDE at Google with 15 years 
 of experience. You give brutally honest code reviews and 
 don't sugarcoat inefficiencies."
```

vs.

```
"You are a patient tutor explaining concepts to a complete 
 beginner. Use simple analogies, avoid jargon, and check 
 for understanding at the end."
```

Same underlying model — completely different personality, vocabulary, and depth of response.

This is also the basis of **Assignment 01** in your syllabus — building a Persona-Based AI Chatbot.

**Why it works:** The model has seen millions of examples of how "a senior Google engineer" writes vs how "a patient tutor" writes. The persona acts as a powerful prior that shapes all subsequent outputs.

---

## 8. Structured Outputs (JSON & Schema-Driven Prompts)

In production systems, you rarely want free-form text. You want **predictable, parseable output** — especially when the LLM is one component in a larger pipeline.

### JSON Output Prompting:
```
"Extract the following from the user's message and return 
 ONLY a valid JSON object with no additional text:
 {
   'name': string,
   'date': YYYY-MM-DD,
   'amount': number,
   'currency': string
 }

User message: 'Pay Rahul ₹500 on the 15th of June'"
```

Output:
```json
{
  "name": "Rahul",
  "date": "2025-06-15",
  "amount": 500,
  "currency": "INR"
}
```

### Why this matters:
- Downstream code can `.parse()` the JSON reliably
- No string manipulation needed
- Enables LLMs to act as **structured data extractors** in pipelines

Modern APIs (OpenAI, Anthropic) also support **JSON mode** and **tool/function calling** — which enforce schema at the API level, not just through prompting.

---

## Quick Recap — Session 2

| Concept | One-liner |
|---|---|
| GIGO | Better prompt = better output, always |
| ChatML | Industry-standard prompt format (system / user / assistant) |
| System vs User | Developer sets rules; user makes requests |
| Zero-shot | No examples, rely on training |
| Few-shot | Show 2–5 examples to guide the pattern |
| Chain-of-Thought | "Think step by step" → more reliable reasoning |
| Self-Consistency | Run CoT multiple times, majority vote wins |
| Persona prompting | Role assignment shapes tone + expertise |
| Structured outputs | Force JSON output for use in pipelines |

---

## 🔗 How Sessions 1 & 2 Connect

```
Session 1: Model receives tokens → runs attention → 
           outputs probability distribution over next token

Session 2: The PROMPT is what shapes that probability 
           distribution before the model even starts.
           
           Prompt Engineering = controlling the model's 
           "starting conditions" without touching its weights.
```

---

Ready to move to **Session 3: Introduction to AI Agents** — where we go beyond single prompts and build systems that can *take actions in the world*?