Perfect choice! Let's build your GenAI foundation from the ground up.

---

# GenAI — Session 1: Introduction to GenAI

---

## 1. What is GPT?

**GPT** stands for **Generative Pre-trained Transformer**.

Let's break that down:

- **Generative** — it *generates* new content (text, code, images, etc.)
- **Pre-trained** — it was trained on a massive corpus of text *before* you ever talk to it
- **Transformer** — it uses a specific neural network architecture called a Transformer (more on this shortly)

> Think of GPT as a student who has read the entire internet, and can now answer your questions by predicting what a good answer looks like — word by word.

---

## 2. "Attention Is All You Need" — The Paper That Started It All

In 2017, Google researchers published a paper titled **"Attention Is All You Need"** — this introduced the **Transformer architecture**, which powers GPT, Claude, Gemini, and virtually every modern LLM.

The key intuition:

> When you read the sentence *"The animal didn't cross the street because **it** was too tired"* — how do you know what **"it"** refers to? You pay **attention** to "animal", not "street".

Transformers do exactly this — they learn which words to **pay attention to** when processing each word. This is called **self-attention**.

Before Transformers, models read text left-to-right sequentially (RNNs). Transformers read the **entire sequence at once** and figure out relationships between all words simultaneously — making them far more powerful and parallelizable.

---

## 3. How Transformers Predict the Next Token

LLMs are fundamentally **next-token predictors**. That's it. Everything else is a consequence of doing this really well at scale.

Here's the flow:

```
Input: "The sky is"
         ↓
   Tokenise the input
         ↓
   Convert tokens → Embeddings
         ↓
   Pass through Transformer layers (Attention + Feed Forward)
         ↓
   Output: probability distribution over all possible next tokens
         ↓
   Sample from distribution → "blue"
```

The model doesn't "know" the answer — it computes the **most probable next token** given everything before it, then repeats.

---

## 4. End-to-End LLM Request → Response Flow

When you type a message to Claude or ChatGPT, here's what actually happens:

```
You type a prompt
       ↓
Text is broken into TOKENS
       ↓
Tokens → Embeddings (numerical vectors)
       ↓
Positional Encoding added (so model knows word order)
       ↓
Passed through N Transformer layers
       ↓
Output layer generates probability scores (logits)
       ↓
Softmax converts logits → probabilities
       ↓
Sampling strategy (Temperature / top-p) picks the next token
       ↓
Token decoded back to text
       ↓
Repeat until <end> token or max_tokens reached
```

---

## 5. Tokenisation & Token Economics

Text is not fed as raw characters. It's split into **tokens** — chunks of characters.

| Text | Tokens |
|---|---|
| "Hello" | `["Hello"]` — 1 token |
| "Unbelievable" | `["Un", "believ", "able"]` — 3 tokens |
| "ChatGPT" | `["Chat", "G", "PT"]` — 3 tokens |

**Why does this matter?**
- APIs charge you **per token** (input + output)
- Models have a **context window** limit in tokens (e.g., 200k tokens for Claude)
- Roughly: **1 token ≈ 0.75 words** in English

> "Token economics" = being smart about how many tokens you use. Verbose prompts cost more and eat into context.

---

## 6. Input Embeddings & Positional Encoding

**Embeddings** — Tokens are converted to vectors (lists of numbers) that capture *meaning*. Similar words end up close together in vector space.

```
"King"  → [0.2, 0.8, -0.1, ...]
"Queen" → [0.2, 0.7, -0.1, ...]   ← very similar!
"Car"   → [-0.9, 0.1, 0.6, ...]   ← very different
```

**Positional Encoding** — Since Transformers process all tokens simultaneously (not sequentially), they need a way to know *word order*. A positional signal is added to each embedding so the model knows token 1 came before token 2.

---

## 7. Softmax, Temperature, Top-p, Max Tokens

These control **how the model picks the next token** from its probability distribution.

**Softmax** converts raw scores (logits) into probabilities that sum to 1:
```
Logits:       [2.0,  1.0,  0.5]
After Softmax: [0.59, 0.24, 0.17]  ← now probabilities
```

**Temperature** — controls randomness:
- `Temperature = 0` → always pick the highest probability token (deterministic, robotic)
- `Temperature = 1` → standard sampling
- `Temperature > 1` → more random/creative outputs

**Top-p (nucleus sampling)** — instead of considering all tokens, only consider the smallest set of tokens whose cumulative probability ≥ p:
- `top_p = 0.9` → only sample from tokens that together make up 90% of the probability mass
- Cuts off the long tail of unlikely/weird tokens

**Max tokens** — simply the hard limit on how many tokens the model will generate in a response.

---

## 8. Inference vs Training

| | Training | Inference |
|---|---|---|
| **What** | Teaching the model by showing billions of examples | Using the trained model to answer a query |
| **When** | Done once (or periodically) by the AI company | Happens every time you send a message |
| **Cost** | Extremely expensive (millions of $) | Cheaper, but scales with usage |
| **You do this?** | Almost never | Every single API call |

---

## 9. Prompt Versioning & Testing

Since the model is fixed after training, **the only thing you control is the prompt**. This makes prompting an engineering discipline:

- Keep track of prompt versions (like code versions)
- Test prompts against a set of expected outputs
- Change one variable at a time when experimenting
- Don't assume a prompt that works on GPT-4 works the same on Claude

---

## Quick Recap — Session 1

| Concept | One-liner |
|---|---|
| GPT | Generative Pre-trained Transformer |
| Transformer | Architecture that uses self-attention to process sequences |
| Next-token prediction | LLMs predict one token at a time |
| Tokenisation | Text → chunks → numbers |
| Embeddings | Numerical meaning-vectors for tokens |
| Positional Encoding | Tells the model word order |
| Softmax | Converts scores → probabilities |
| Temperature / Top-p | Control creativity/randomness |
| Inference vs Training | Using the model vs building it |

---

Want me to do a **quick quiz** on Session 1 before we move to Session 2 (Prompt Engineering), or shall we proceed straight to the next session?