# Enterprise AI Agent Architecture Handbook
### *A Visual Reference for the Observability & RCA Platform Engineer*

> **Running Example Throughout This Handbook:**
> A user asks: *"Why has MarketID ABC stopped processing trades since 10:30?"*
> Every concept is explained through this lens.

---

# Table of Contents

| # | Chapter | Core Question It Answers |
|---|---------|--------------------------|
| 1 | LLM Fundamentals | What is the brain behind the agent? |
| 2 | Tokens | How does the LLM read and write? |
| 3 | Context Windows | How much can the LLM hold in memory at once? |
| 4 | Prompt Engineering | How do you talk to an LLM effectively? |
| 5 | Context Engineering | How do you manage what goes into the LLM's workspace? |
| 6 | Structured Outputs | How do you get reliable, parseable answers from an LLM? |
| 7 | Function Calling | How does an LLM decide to use an external tool? |
| 8 | Tool Calling | How does an agent actually execute actions? |
| 9 | MCP | How do agents connect to the outside world in a standard way? |
| 10 | Intent | What does the user actually want? |
| 11 | Entity Extraction | Who or what is the user talking about? |
| 12 | Entity Resolution | Mapping fuzzy names to real system IDs |
| 13 | Semantic Layer | Building a shared vocabulary between human and machine |
| 14 | Planner | How does the agent decide what to do next? |
| 15 | Routing | How does the agent pick the right tool or agent? |
| 16 | Capabilities | What can the agent actually do? |
| 17 | Tool Registry | Where are all the tools listed? |
| 18 | Capability Registry | What can each agent do, and who decides? |
| 19 | State | How does the agent remember what it has done so far? |
| 20 | Memory | How does the agent remember across sessions? |
| 21 | RAG | How does the agent look up knowledge it wasn't trained on? |
| 22 | Knowledge Graphs | How does the agent understand relationships between things? |
| 23 | Multi-Agent Systems | When one agent is not enough |
| 24 | Workflow Orchestration | Who directs the agents? |
| 25 | LangGraph Concepts | State machines for agent workflows |
| 26 | Guardrails | How do you keep the agent from going wrong? |
| 27 | Evaluation | How do you know if the agent is working well? |
| 28 | Agent Observability | How do you monitor the agent itself? |
| 29 | Security | How do you keep agents safe in enterprise? |
| 30 | Human-in-the-Loop | When should a human take over? |
| 31 | Production Architecture | The complete picture |

---

# Chapter 1: LLM Fundamentals

## 1. Definition

A **Large Language Model (LLM)** is a software system trained on vast amounts of text. It learns statistical patterns — which words, phrases, and ideas tend to follow each other — and uses those patterns to generate coherent, contextually relevant text in response to an input.

It is not a search engine. It does not look things up. It **generates** based on learned patterns.

Think of it as a very well-read colleague who has absorbed millions of documents and can synthesize answers — but who cannot open a browser, cannot query your database, and whose knowledge has a cutoff date.

---

## 2. Why It Exists

Before LLMs, making a computer understand *"Why has MarketID ABC stopped processing trades since 10:30?"* required:

- A rigid query parser that understood only specific grammar
- Explicit rules mapping every possible phrasing to a database query
- Separate NLP pipelines for intent detection, entity recognition, disambiguation

This was fragile. Change one word and the system breaks.

LLMs solve this by understanding **natural language in all its messiness** — abbreviations, typos, domain jargon, implied context — and producing coherent, context-aware responses.

**Without LLMs:** You'd need a team of engineers to hardcode every possible question a trader could ask.
**With LLMs:** The same model handles "MarketID ABC is down since 10:30", "why did ABC trades stop?", "ABC market not processing – what happened?" — all naturally.

---

## 3. Real-World Analogy

```
┌─────────────────────────────────────────────────────────────┐
│                     SENIOR IT ENGINEER                       │
│                                                             │
│  • Has worked on 20+ systems over 15 years                  │
│  • Knows how Kafka, Loki, Prometheus behave                 │
│  • Can't directly SSH into your server (yet)                │
│  • Can reason about symptoms and suggest next steps          │
│  • Needs TOOLS (runbooks, dashboards) to actually fix things │
└─────────────────────────────────────────────────────────────┘
```

An LLM is that senior engineer's *brain* — without arms. It can reason, synthesize, and plan. But it needs tools to act.

---

## 4. Observability Example

When a user types:
> *"Why has MarketID ABC stopped processing trades since 10:30?"*

The LLM's role is to:
1. Understand this is a root cause investigation, not a how-to question
2. Recognize "MarketID ABC" and "10:30" as important anchors
3. Reason about what systems could cause a trade stoppage
4. Generate a coherent investigation plan

The LLM does **not** query Kafka or Loki by itself. It reasons. Other layers act.

---

## 5. Visual Workflow

### How an LLM Processes Input

```
┌─────────────────────────────────────────────────────────┐
│                      INPUT                              │
│   "Why has MarketID ABC stopped processing trades       │
│    since 10:30?"                                        │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│                   TOKENIZATION                          │
│   ["Why", " has", " Market", "ID", " ABC", " stopped", │
│    " processing", " trades", " since", " 10", ":30", "?"]│
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│              PATTERN MATCHING (INFERENCE)               │
│                                                         │
│   Model computes: given this sequence of tokens,        │
│   what is the most likely next token? And the next?     │
│   And the next? Until the response is complete.         │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│                      OUTPUT                             │
│   "This appears to be a trade processing outage.        │
│    I should check: Kafka consumer lag for MarketID ABC, │
│    error logs in Loki around 10:30, and Prometheus      │
│    metrics for the processing service."                 │
└─────────────────────────────────────────────────────────┘
```

### What an LLM Is vs. What It Is Not

```
┌──────────────────────────────┬──────────────────────────────────┐
│        LLM IS                │        LLM IS NOT                │
├──────────────────────────────┼──────────────────────────────────┤
│ A reasoning engine           │ A search engine                  │
│ A language generator         │ A database                       │
│ A pattern synthesizer        │ A rule engine                    │
│ A planner                    │ An executor                      │
│ A summarizer                 │ A real-time data source          │
│ An ambiguity resolver        │ Deterministic (always same out)  │
└──────────────────────────────┴──────────────────────────────────┘
```

### LLM in the Agent Stack (Preview)

```
┌───────────────────────────────────────────────────────────┐
│                     AGENT SYSTEM                          │
│                                                           │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐              │
│   │  User   │───▶│   LLM   │───▶│  Tools  │              │
│   └─────────┘    │ (Brain) │    │ (Arms)  │              │
│                  └────┬────┘    └────┬────┘              │
│                       │              │                    │
│                       ▼              ▼                    │
│                  ┌─────────┐    ┌─────────┐              │
│                  │ Planner │    │  Loki   │              │
│                  │  &      │    │ Kafka   │              │
│                  │ Reasoner│    │ Grafana │              │
│                  └─────────┘    └─────────┘              │
└───────────────────────────────────────────────────────────┘
```

---

## 6. Step-by-Step Walkthrough

**User types:** *"Why has MarketID ABC stopped processing trades since 10:30?"*

**Step 1 — Text arrives at the LLM**
The entire question is passed as input text.

**Step 2 — Tokenization**
The model converts the text into tokens (covered in Chapter 2).

**Step 3 — Attention**
The model identifies which words are most important and how they relate. "MarketID ABC" and "10:30" and "stopped processing trades" get high attention weights.

**Step 4 — Generation**
The model generates a response token by token. Each token is selected based on probability given everything before it.

**Step 5 — Output**
The output might be: a plan, a tool call, a question, or a direct answer — depending on how the system is configured (covered in Chapters 4–8).

---

## 7. Enterprise Design

Most enterprise AI platforms use LLMs as the **reasoning core** and build a scaffolding around them:

```
┌──────────────────────────────────────────────────────────┐
│                  ENTERPRISE LLM TOPOLOGY                 │
│                                                          │
│  ┌─────────────┐         ┌──────────────┐               │
│  │  API Gateway│────────▶│  LLM Provider│               │
│  │  (Rate limit│         │  (OpenAI /   │               │
│  │   Auth, Log)│         │   Anthropic / │               │
│  └─────────────┘         │   Azure OAI) │               │
│                          └──────┬───────┘               │
│                                 │                        │
│              ┌──────────────────┤                        │
│              │                  │                        │
│              ▼                  ▼                        │
│       ┌──────────┐      ┌──────────────┐                │
│       │  Prompt  │      │   Response   │                │
│       │ Templates│      │   Parsers    │                │
│       └──────────┘      └──────────────┘                │
└──────────────────────────────────────────────────────────┘
```

**Key design decisions:**
- Which LLM provider? (Latency, cost, data residency)
- Self-hosted vs. API? (Morgan Stanley would likely require private deployment)
- Which model size? Bigger ≠ always better for structured tasks

---

## 8. Common Mistakes

| Mistake | Why it Happens | Real Consequence |
|---------|---------------|-----------------|
| Treating LLM output as ground truth | Impressive fluency creates false confidence | Agent acts on hallucinated Kafka consumer group names |
| Using LLM for exact lookup tasks | It feels natural to ask the LLM directly | LLM guesses a MarketID instead of querying MongoDB |
| No output validation | Developer trusts the model | Downstream tool gets malformed JSON, crashes silently |
| Sending too much raw data to LLM | "More context = better" assumption | Context window overflow, high cost, slow response |
| One giant prompt for everything | Easier to write, hard to maintain | Agent fails unpredictably on edge cases |

---

## 9. Framework Mapping

| Concept | LangGraph | OpenAI SDK | Claude SDK | Semantic Kernel |
|---------|-----------|------------|------------|-----------------|
| LLM invocation | Node in graph calls LLM | `client.responses.create()` | `client.messages.create()` | `kernel.invoke()` |
| Model selection | Configured per node | `model` parameter | `model` parameter | `AIService` config |
| Output handling | Node return value | Response object | Content blocks | `FunctionResult` |

---

## 10. Summary

**Key Ideas:**
- An LLM is a reasoning engine, not a data retrieval system
- It generates text based on learned patterns, not rule lookups
- It needs tools to interact with the real world
- Its output must be validated before being trusted

**Mental Model:**
> The LLM is the brain of a senior engineer who has read every runbook, every post-mortem, every architecture document — but is sitting in a room with no computer access. You pass them notes. They reason. They tell you what to check. Other systems do the actual checking.

**Interview Questions:**
1. What is the difference between a language model and a search engine?
2. Why can't an LLM query your Kafka cluster directly?
3. What does "hallucination" mean in the context of an LLM?

**Self-Check Questions:**
- Can an LLM reliably return the exact Kafka consumer lag for MarketID ABC? Why or why not?
- If the LLM says "the error started at 10:28", should you trust that without verification?

**Practical Exercise:**
Send this prompt to any LLM API: *"Why has MarketID ABC stopped processing trades since 10:30?"* — with no system prompt, no tools, no context. Observe the response. Note: it will hallucinate specific technical details. This is your baseline for understanding why everything else in this handbook exists.

---

# Chapter 2: Tokens

## 1. Definition

A **token** is the smallest unit of text that an LLM processes. Tokens are not always words. They can be:
- A full word: `trade` → 1 token
- Part of a word: `processing` → 1 token, but `MarketID` might be 2 (`Market` + `ID`)
- A punctuation mark: `:` → 1 token
- A number: `10` → 1 token, `10:30` → 2–3 tokens

The LLM never sees raw text. It sees a sequence of integer IDs, one per token.

---

## 2. Why It Exists

Computers cannot process arbitrary-length strings efficiently. Tokenization converts text into a fixed, manageable vocabulary of integer IDs (typically 50,000–100,000 unique tokens).

This matters to you for three practical reasons:

1. **Cost** — LLM API pricing is per token (input + output). A verbose prompt costs more.
2. **Speed** — More tokens = slower inference.
3. **Limits** — The context window (Chapter 3) is measured in tokens. Overflow = data loss.

---

## 3. Real-World Analogy

```
┌─────────────────────────────────────────────────────────┐
│                   POSTAL SYSTEM                         │
│                                                         │
│  You can't ship "a house". You break it into:           │
│  - Boxes (sentences)                                    │
│  - Items (words)                                        │
│  - Labels (tokens/IDs)                                  │
│                                                         │
│  The postal system only understands label numbers,      │
│  not what's inside. The LLM only understands token IDs, │
│  not raw text.                                          │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Observability Example

Your user's question:
> *"Why has MarketID ABC stopped processing trades since 10:30?"*

Approximate tokenization:

```
┌──────────────────────────────────────────────────────────┐
│  Token Breakdown                                         │
│                                                          │
│  "Why"         → [10919]                                 │
│  " has"        → [508]                                   │
│  " Market"     → [9134]                                  │
│  "ID"          → [926]                                   │
│  " ABC"        → [17844]                                 │
│  " stopped"    → [10397]                                 │
│  " processing" → [8204]                                  │
│  " trades"     → [28833]                                 │
│  " since"      → [2533]                                  │
│  " 10"         → [838]                                   │
│  ":30"         → [25]  + [966]                           │
│  "?"           → [30]                                    │
│                                                          │
│  Total: ~13 tokens for this question                     │
└──────────────────────────────────────────────────────────┘
```

Now consider: a Loki log chunk might be 2,000 tokens. A Kafka consumer group list might be 500 tokens. A Prometheus metric dump might be 3,000 tokens. Stack three tool results together and you've consumed significant context window budget before the LLM even begins reasoning.

---

## 5. Visual Workflow

### Token Flow

```
Raw Text Input
      │
      ▼
┌─────────────┐
│  Tokenizer  │  Converts text → integer IDs
│  (BPE /     │  Vocabulary: ~100k entries
│   WordPiece)│
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────────────┐
│   Token Sequence                            │
│   [10919, 508, 9134, 926, 17844, 10397 ...] │
└──────┬──────────────────────────────────────┘
       │
       ▼
┌─────────────┐
│    LLM      │  Processes token IDs
│  (Inference)│  Generates next token ID
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Detokenizer│  Converts integer IDs → text
└──────┬──────┘
       │
       ▼
Output Text Response
```

### Token Budget Planning for RCA Query

```
┌────────────────────────────────────────────────────────────┐
│              CONTEXT WINDOW BUDGET (128k tokens)           │
│                                                            │
│  ████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░   │
│  │                                                         │
│  ├── System Prompt (instructions)     ~2,000 tokens        │
│  ├── User Question                      ~20 tokens         │
│  ├── Conversation History             ~5,000 tokens        │
│  ├── Loki Logs (retrieved)           ~10,000 tokens        │
│  ├── Prometheus Metrics               ~3,000 tokens        │
│  ├── Kafka State                      ~2,000 tokens        │
│  ├── MongoDB Trade Records            ~8,000 tokens        │
│  └── Reserved for LLM Output         ~5,000 tokens        │
│                                                            │
│  Total Used: ~35,020 / 128,000 tokens                      │
│  Remaining:  ~92,980 tokens                                │
└────────────────────────────────────────────────────────────┘
```

### Token Cost Comparison: Verbose vs. Compact Context

```
                     VERBOSE APPROACH
┌──────────────────────────────────────────────────────────┐
│  Raw Loki dump: 50,000 log lines                         │
│  = ~150,000 tokens  ← EXCEEDS CONTEXT WINDOW             │
│  Cost: $4.50 per query (at $0.03/1k tokens)              │
│  Result: Context overflow, logs truncated                │
└──────────────────────────────────────────────────────────┘

                     COMPACT APPROACH
┌──────────────────────────────────────────────────────────┐
│  Filtered Loki: 200 relevant log lines around 10:30      │
│  = ~6,000 tokens  ← FITS COMFORTABLY                     │
│  Cost: $0.18 per query                                   │
│  Result: Precise, full context preserved                 │
└──────────────────────────────────────────────────────────┘
```

---

## 6. Step-by-Step Walkthrough

**Step 1** — User submits: *"Why has MarketID ABC stopped processing trades since 10:30?"*

**Step 2** — The tokenizer (part of the LLM API) converts the question to ~13 token IDs.

**Step 3** — The system prompt (your instructions to the LLM about how to behave as an RCA agent) adds ~2,000 tokens.

**Step 4** — Retrieved tool results (Loki logs, Prometheus metrics) add ~15,000 tokens.

**Step 5** — Total input = ~17,013 tokens. This is passed to the LLM for inference.

**Step 6** — The LLM generates output tokens one at a time until it completes its response (~500–2,000 tokens for a typical RCA answer).

**Step 7** — The output is detokenized back to readable text.

---

## 7. Enterprise Design

**Token optimization is a first-class concern in production:**

| Strategy | Description | Applied to RCA Platform |
|----------|-------------|------------------------|
| Summarization | Compress tool results before injecting | Summarize 5,000 Loki lines to 500-token digest |
| Chunking | Split large data into retrievable pieces | Store log chunks in vector DB, retrieve top-k |
| Filtering | Only send relevant data | Filter Prometheus metrics to MarketID ABC only |
| Caching | Cache repeated tool results | Cache MarketID→service mappings |
| Truncation policy | Decide what to cut first | Cut oldest logs before newest |

---

## 8. Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| Dumping entire Loki log file into prompt | Context overflow; critical logs at the end get cut |
| Not tracking token usage per query | Runaway costs; model throttling |
| Ignoring tokenization for non-English text | Non-Latin scripts tokenize less efficiently (more tokens per word) |
| Assuming 1 word = 1 token in budget estimates | Off-by-2x errors in capacity planning |

---

## 9. Framework Mapping

| Framework | Token Handling |
|-----------|---------------|
| LangGraph | Node-level token tracking via `RunnableConfig` |
| OpenAI SDK | `usage` field in response: `prompt_tokens`, `completion_tokens` |
| Claude SDK | `usage.input_tokens`, `usage.output_tokens` |
| Semantic Kernel | Token counting via `TokenCounter` utilities |

---

## 10. Summary

**Key Ideas:**
- Tokens are the atomic unit of LLM processing — not words, not characters
- Token limits (context window) are a hard ceiling, not a soft guideline
- Cost and speed scale linearly with token count
- Compressing tool results before injecting them is essential for production

**Mental Model:**
> The context window is a fixed-size whiteboard. Every token is a word written on it. Once it's full, you must erase something to write more. The LLM can only reason about what's on the whiteboard right now.

**Self-Check Questions:**
- Why would sending raw Loki logs be problematic for a 128k context window?
- How would you estimate the token cost of one RCA investigation?

---

# Chapter 3: Context Windows

## 1. Definition

The **context window** is the total amount of text (measured in tokens) that an LLM can see at one time — input and output combined.

Everything the LLM knows about your current conversation, all retrieved data, all instructions — must fit inside this window. What's outside it doesn't exist for the LLM.

Current practical limits:
- GPT-4o: 128k tokens
- Claude 3.5 Sonnet: 200k tokens
- Gemini 1.5 Pro: 1M tokens (though quality degrades with very long contexts)

---

## 2. Why It Exists

LLMs use a mechanism called **attention** — every token can "attend to" every other token in the window. This is computationally expensive. The cost grows quadratically with window size. A 200k token window is not 2x the cost of 100k — it's roughly 4x.

Context windows exist as a hard engineering boundary between what is possible and what is practical.

For your RCA platform, this means:
- You cannot dump everything you know about your systems into one query
- You must be **selective** about what evidence you retrieve and inject
- You must design **retrieval strategies** (covered in Chapter 21 — RAG)

---

## 3. Real-World Analogy

```
┌─────────────────────────────────────────────────────────┐
│              THE DETECTIVE'S CASE BOARD                 │
│                                                         │
│  A detective can only pin so many clues to their        │
│  whiteboard at once. When the board is full, they must  │
│  take something off to add something new.               │
│                                                         │
│  They can only solve the case using what's on the       │
│  board right now. What's in the filing cabinet          │
│  (your MongoDB, Loki, etc.) doesn't help               │
│  unless someone retrieves it and pins it up.            │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Observability Example

For the query *"Why has MarketID ABC stopped processing trades since 10:30?"*, your agent will retrieve:

```
┌─────────────────────────────────────────────────────────────────┐
│                    CONTEXT WINDOW IN USE                        │
│                                                                 │
│  ┌─── SYSTEM PROMPT ─────────────────────────────────────┐     │
│  │  "You are an RCA agent. When investigating trade       │     │
│  │   outages, always check: Kafka lag, error logs,        │     │
│  │   service health. MarketID maps to internal service    │     │
│  │   codes via the registry..."                           │     │
│  └───────────────────────────────────────────────────────┘     │
│                              ~2k tokens                         │
│  ┌─── USER MESSAGE ──────────────────────────────────────┐     │
│  │  "Why has MarketID ABC stopped processing trades       │     │
│  │   since 10:30?"                                        │     │
│  └───────────────────────────────────────────────────────┘     │
│                              ~20 tokens                         │
│  ┌─── TOOL RESULT: LOKI LOGS ────────────────────────────┐     │
│  │  [2024-01-15 10:29:45] ERROR TradeProcessor: ...       │     │
│  │  [2024-01-15 10:30:01] FATAL Connection refused...     │     │
│  │  [2024-01-15 10:30:03] ERROR Kafka consumer timeout... │     │
│  └───────────────────────────────────────────────────────┘     │
│                              ~8k tokens                         │
│  ┌─── TOOL RESULT: PROMETHEUS ───────────────────────────┐     │
│  │  kafka_consumer_lag{market="ABC"}: 45000              │     │
│  │  trade_processor_up{market="ABC"}: 0                  │     │
│  └───────────────────────────────────────────────────────┘     │
│                              ~1k tokens                         │
│  ┌─── RESERVED FOR OUTPUT ───────────────────────────────┐     │
│  │  [Space for LLM response]                              │     │
│  └───────────────────────────────────────────────────────┘     │
│                              ~3k tokens                         │
│                                                                 │
│  TOTAL: ~14k / 128k tokens used                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Visual Workflow

### What Happens When Context Overflows

```
Attempt to fit everything:

[System Prompt 2k] + [History 20k] + [All Loki Logs 150k] + [Metrics 5k]
= 177,000 tokens  ──▶  EXCEEDS 128k LIMIT

What the LLM actually sees (truncation from the right):

[System Prompt 2k] + [History 20k] + [PARTIAL Loki Logs 106k]
                                      ▲
                                      └── The most recent, most relevant
                                          logs get CUT because they're
                                          at the END of the dump
```

### Context Window Management Strategy

```
                    EVIDENCE RETRIEVAL PIPELINE
                    
User Query
    │
    ▼
┌───────────────────────────────────────────────────────────┐
│                  RELEVANCE FILTERING                      │
│                                                           │
│  ┌──────────┐   Filter by:     ┌──────────────────────┐  │
│  │  Loki    │──▶ Time window ──▶ Top 200 log lines     │  │
│  │  (50k    │   MarketID ABC    ~6k tokens              │  │
│  │   lines) │                  └──────────────────────┘  │
│  └──────────┘                                            │
│                                                           │
│  ┌──────────┐   Filter by:     ┌──────────────────────┐  │
│  │Prometheus│──▶ market=ABC ───▶ Relevant metrics only │  │
│  │ (10k     │   10:00–11:00    ~1k tokens              │  │
│  │  series) │                  └──────────────────────┘  │
│  └──────────┘                                            │
└───────────────────────────────────────────────────────────┘
    │
    ▼
Context Window (14k tokens used, 114k remaining for further steps)
```

### Multi-Turn Context Growth

```
Turn 1:  [System 2k] [User Q 0.1k] [Tool Results 10k] [Response 1k]
         = 13.1k tokens

Turn 2:  [System 2k] [Turn1 13.1k] [User Q2 0.1k] [Tool Results 8k] [Response 1k]
         = 24.2k tokens

Turn 5:  [System 2k] [Turns1-4 50k] [User Q5 0.1k] [Tool Results 8k] [Response 1k]
         = 61.1k tokens  ← Watch this carefully

Turn 10: Context window pressure becomes real. Must summarize history.
```

---

## 6. Step-by-Step Walkthrough

**Step 1** — Agent begins a new RCA session for the MarketID ABC query.

**Step 2** — System prompt is loaded: ~2,000 tokens. This defines agent behavior.

**Step 3** — User question is appended: ~20 tokens.

**Step 4** — Agent calls Loki tool. Returns 200 filtered log lines: ~6,000 tokens. These are appended to the context.

**Step 5** — Agent calls Prometheus tool. Returns 10 key metrics: ~500 tokens. Appended.

**Step 6** — LLM now sees: [2k + 20 + 6k + 500] = ~8,520 tokens of input.

**Step 7** — LLM generates its analysis: ~800 tokens of output.

**Step 8** — If user asks a follow-up, the entire previous exchange (8,520 + 800 = 9,320 tokens) plus new content gets appended. Context grows.

---

## 7. Enterprise Design

| Pattern | Description | Trade-off |
|---------|-------------|-----------|
| **Sliding Window** | Keep only the last N turns in context | Loses earlier findings |
| **Summarization** | Compress old turns into a summary | Loses precision |
| **External Memory** | Store findings in DB, retrieve when needed | Extra retrieval latency |
| **Stateless per Query** | Each RCA query starts fresh | No cross-session learning |
| **Hierarchical Context** | Summary + full recent turns | Best quality, more complex |

For an RCA platform, **hierarchical context** works best: compress older investigation steps into a "findings so far" summary, keep the last 2–3 tool results in full.

---

## 8. Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| Injecting full log files | Context overflow; oldest logs cut first |
| Not tracking context size during multi-turn | Sudden failure at turn 8 after 7 successful turns |
| Assuming truncation is safe | Most frameworks truncate from the right — you lose recent data |
| No context budget planning | Each tool result competes for space without a budget |

---

## 9. Summary

**Key Ideas:**
- The context window is the LLM's working memory — finite and precious
- What isn't in the context doesn't exist for the LLM
- Context grows with every turn — plan for multi-turn degradation
- Filter and compress evidence before injecting it

**Mental Model:**
> The context window is RAM, not disk. Fast, powerful — but limited. Your job is to load only what the CPU (LLM) needs right now to reason. Everything else stays on disk (your databases) and gets paged in on demand.

---

# Chapter 4: Prompt Engineering

## 1. Definition

**Prompt engineering** is the practice of crafting the text you send to an LLM to maximize the quality, consistency, and usefulness of its output.

A prompt is everything the LLM receives as input — the instructions, the context, the examples, the user question. The quality of the prompt directly determines the quality of the output.

---

## 2. Why It Exists

LLMs are remarkably sensitive to how you phrase things. The same question worded differently can produce wildly different results:

- *"Check if there's a problem"* → Vague, produces vague output
- *"You are an SRE investigating a trade processing outage. Analyze the following Kafka consumer lag and Loki error logs. Output: (1) root cause hypothesis, (2) confidence level, (3) next investigation step."* → Precise, produces actionable output

Without prompt engineering, LLMs produce plausible-sounding text. With it, they produce reliable, structured, domain-specific analysis.

---

## 3. Real-World Analogy

```
┌─────────────────────────────────────────────────────────┐
│                   BRIEFING A CONSULTANT                 │
│                                                         │
│  BAD BRIEF:                                             │
│  "Look at our trading system and tell us what's wrong." │
│  → Consultant writes a 20-page generic report           │
│                                                         │
│  GOOD BRIEF:                                            │
│  "Analyze MarketID ABC trade processing downtime from   │
│   10:30 today. Focus on: Kafka consumer lag, connection │
│   errors in the trade processor service, and upstream   │
│   dependency availability. Provide: root cause          │
│   (probable/confirmed), evidence, remediation steps."   │
│  → Consultant delivers exactly what you need            │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Observability Example

**Poorly Engineered Prompt:**
```
System: You are a helpful assistant.
User: Why has MarketID ABC stopped processing trades since 10:30?
```
*Result: Generic answer. May suggest rebooting servers. Ignores your actual stack.*

**Well-Engineered Prompt:**
```
System: You are an SRE agent for a financial trading platform.
You investigate trade processing outages by correlating:
- Kafka consumer lag (source: Prometheus)
- Application errors (source: Loki)
- Service health (source: Grafana)
- Trade records (source: MongoDB)

When analyzing an outage, always:
1. Identify the time of first error
2. Correlate across at least 2 data sources before forming a hypothesis
3. Distinguish between root cause and symptoms
4. Output confidence level: [HIGH / MEDIUM / LOW]
5. Suggest the next concrete investigation step

User: Why has MarketID ABC stopped processing trades since 10:30?

Context:
[Loki logs injected here]
[Prometheus metrics injected here]
```
*Result: Structured RCA with source citations, confidence rating, next steps.*

---

## 5. Visual Workflow

### Anatomy of a Well-Engineered Prompt

```
┌──────────────────────────────────────────────────────────────────┐
│                        FULL PROMPT STRUCTURE                     │
│                                                                  │
│  ┌── SYSTEM PROMPT ─────────────────────────────────────────┐   │
│  │  [1] ROLE          "You are an SRE agent..."             │   │
│  │  [2] DOMAIN        "...for a financial trading platform"  │   │
│  │  [3] DATA SOURCES  "You have access to: Loki, Prometheus" │   │
│  │  [4] BEHAVIOR      "Always correlate 2+ sources..."      │   │
│  │  [5] OUTPUT FORMAT "Respond in JSON with fields: ..."    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌── FEW-SHOT EXAMPLES (optional) ─────────────────────────┐   │
│  │  User: "Why did MarketID XYZ fail at 09:15?"             │   │
│  │  Agent: { "root_cause": "Kafka partition leadership...", │   │
│  │           "confidence": "HIGH", "evidence": [...] }      │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌── RETRIEVED CONTEXT ─────────────────────────────────────┐   │
│  │  Loki logs: [filtered, relevant lines]                    │   │
│  │  Prometheus: [relevant metrics]                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌── USER MESSAGE ──────────────────────────────────────────┐   │
│  │  "Why has MarketID ABC stopped processing trades          │   │
│  │   since 10:30?"                                           │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### Prompt Engineering Techniques

```
┌──────────────────────────────────────────────────────────┐
│              PROMPT ENGINEERING TOOLKIT                  │
│                                                          │
│  ROLE ASSIGNMENT                                         │
│  "You are an expert SRE..."                              │
│  ─────────────────────────────────────────────────────   │
│  EXPLICIT OUTPUT FORMAT                                  │
│  "Respond only in JSON: {root_cause, confidence, ...}"   │
│  ─────────────────────────────────────────────────────   │
│  CHAIN OF THOUGHT                                        │
│  "Think step by step before concluding"                  │
│  ─────────────────────────────────────────────────────   │
│  FEW-SHOT EXAMPLES                                       │
│  Show 1–3 input/output pairs before the real question    │
│  ─────────────────────────────────────────────────────   │
│  CONSTRAINTS                                             │
│  "Do not guess. If evidence is insufficient, say so."   │
│  ─────────────────────────────────────────────────────   │
│  NEGATIVE EXAMPLES                                       │
│  "Do NOT output markdown. Do NOT include apologies."     │
└──────────────────────────────────────────────────────────┘
```

### Impact of Prompt Quality on RCA Output

```
  Prompt Quality          Output Quality
        │                       │
   VAGUE│  "You are helpful"    │ Generic suggestions, wrong stack
        │                       │
  BASIC │  Role + question      │ Better, but inconsistent format
        │                       │
   GOOD │  Role + format + data │ Consistent structure, relevant findings
        │                       │
OPTIMAL │  Role + format +      │ Production-ready: structured, cited,
        │  examples + data +    │ confidence-rated, source-referenced
        │  constraints          │
        ▼                       ▼
```

---

## 6. Step-by-Step Walkthrough

**Step 1** — Define the LLM's role in your system: SRE agent, not a general assistant.

**Step 2** — Define what data sources it has access to. This primes it to ask for relevant tools.

**Step 3** — Define expected reasoning behavior: correlate before concluding, cite sources.

**Step 4** — Define output format explicitly: JSON with specific fields.

**Step 5** — Add 1–2 worked examples of past RCA queries and their ideal outputs.

**Step 6** — At runtime, inject the retrieved Loki logs and Prometheus metrics into the prompt.

**Step 7** — Append the user's question.

**Step 8** — Send to LLM. Parse the JSON output. Validate against schema.

---

## 7. Enterprise Design

| Pattern | Use Case | Applied to RCA Platform |
|---------|----------|------------------------|
| **System Prompt Templates** | Reusable base instructions | One template per investigation type (outage, latency, data quality) |
| **Dynamic Prompt Assembly** | Inject runtime data | Fill in MarketID, time range, retrieved logs at query time |
| **Prompt Versioning** | Track changes over time | Git-controlled `.txt` or `.jinja2` files per prompt |
| **A/B Prompt Testing** | Measure output quality | Compare two prompt versions on 100 real queries |
| **Prompt Caching** | Reduce cost | Cache the static system prompt prefix |

---

## 8. Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| No output format specified | LLM returns markdown prose; parsing breaks |
| Overcrowded system prompt | LLM loses track of instructions buried in paragraphs |
| No negative constraints | LLM apologizes, hedges, includes irrelevant explanations |
| Changing prompt without versioning | Regression: yesterday's RCA quality was better; can't reproduce |
| Assuming one prompt works for all query types | Latency queries need different reasoning than outage queries |

---

## 9. Summary

**Key Ideas:**
- Prompts are the primary way to control LLM behavior
- Structure beats verbosity: clear sections outperform long paragraphs
- Format constraints and worked examples dramatically improve consistency
- Prompt engineering is an iterative, measurable discipline — not art

**Mental Model:**
> Writing a system prompt is like writing a job description + training manual for a new analyst. The more precise your expectations, examples, and constraints, the better the output.

---

# Chapter 5: Context Engineering

## 1. Definition

**Context engineering** is the discipline of deciding *what information* to put into the LLM's context window, *when* to put it in, and *how* to format it — so the LLM can reason effectively without exceeding its limits.

Prompt engineering focuses on *how you phrase instructions*. Context engineering focuses on *what evidence you give the LLM to work with*.

If prompt engineering is writing the brief, context engineering is assembling the case file.

---

## 2. Why It Exists

The LLM's reasoning quality is bounded by the quality of its input context. Give it noisy, irrelevant, or truncated data and it will produce noisy, irrelevant, or incomplete analysis.

In your RCA platform, this is the difference between:
- Giving the LLM all 50,000 Loki log lines → noise, truncation, wrong focus
- Giving the LLM the 200 most relevant log lines around 10:30 for MarketID ABC → precise, evidence-based analysis

Context engineering is the discipline that bridges raw data sources (Loki, Kafka, Prometheus) and the LLM's reasoning capacity.

---

## 3. Real-World Analogy

```
┌─────────────────────────────────────────────────────────┐
│             PREPARING FOR A BOARD PRESENTATION          │
│                                                         │
│  You have 10GB of operational data.                     │
│  The board has 30 minutes.                              │
│                                                         │
│  BAD APPROACH: Print all 10GB and dump it on the table  │
│  GOOD APPROACH: Select the 5 most relevant charts,      │
│    summarize the key trend, highlight the anomaly,      │
│    present it in a format they can absorb in 30 minutes │
│                                                         │
│  Context engineering = preparing those 5 charts.        │
│  The LLM = the board. Its context window = 30 minutes.  │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Observability Example

For the query *"Why has MarketID ABC stopped processing trades since 10:30?"*, context engineering decides:

```
┌──────────────────────────────────────────────────────────────┐
│                  CONTEXT ENGINEERING DECISIONS               │
│                                                              │
│  FROM LOKI:                                                  │
│  ✗ All logs from all services for all day                    │
│  ✓ ERROR/FATAL logs only, service=trade-processor,           │
│    market=ABC, time=10:15–10:45, max 200 lines               │
│                                                              │
│  FROM PROMETHEUS:                                            │
│  ✗ All 10,000 metric series                                  │
│  ✓ kafka_consumer_lag{market="ABC"},                         │
│    trade_processor_up{market="ABC"},                         │
│    time=10:00–11:00, 1-minute resolution                     │
│                                                              │
│  FROM MONGODB:                                               │
│  ✗ Full trade records collection                             │
│  ✓ Last 50 failed trades for MarketID ABC after 10:30        │
│                                                              │
│  FROM KAFKA:                                                 │
│  ✗ All consumer group offsets                                │
│  ✓ Consumer group state for MarketID-ABC-processor group     │
└──────────────────────────────────────────────────────────────┘
```

---

## 5. Visual Workflow

### Context Assembly Pipeline

```
User Query: "Why has MarketID ABC stopped processing trades since 10:30?"
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│                 CONTEXT ENGINEERING LAYER                │
│                                                          │
│  Step 1: PARSE QUERY                                     │
│  Extract: entity=MarketID_ABC, time=10:30, type=outage   │
│                │                                         │
│                ▼                                         │
│  Step 2: DETERMINE RELEVANT SOURCES                      │
│  Outage type → Kafka + Loki + Prometheus                 │
│                │                                         │
│                ▼                                         │
│  Step 3: FETCH WITH FILTERS                              │
│  Loki: market=ABC, severity=ERROR+, ±15min of 10:30      │
│  Prometheus: market=ABC metrics, 10:00–11:00             │
│  Kafka: consumer group ABC-processor, current state      │
│                │                                         │
│                ▼                                         │
│  Step 4: RANK AND TRIM                                   │
│  Score each log line by relevance. Keep top 200.         │
│  Summarize Prometheus: "Lag went from 0 to 45k at 10:31" │
│                │                                         │
│                ▼                                         │
│  Step 5: FORMAT FOR LLM                                  │
│  Structure context with clear section headers            │
│  Label each source: "## Loki Logs (10:15–10:45)"         │
└──────────────────────────────────────────────────────────┘
    │
    ▼
Assembled Context → LLM → RCA Analysis
```

### Context Quality Spectrum

```
         NOISE                                    SIGNAL
            │                                       │
            ▼                                       ▼
┌──────────────────────────────────────────────────────┐
│  Raw dump        Filtered      Summarized   Curated  │
│  50k log lines → 2k lines   → Key events  → 200 lines│
│                                                      │
│  LLM Quality:  Poor      Acceptable    Good  Excellent│
│  Token Cost:   High      Medium        Low   Very Low│
└──────────────────────────────────────────────────────┘
```

### The Context Engineering Contract

```
┌──────────────────────────────────────────────────────────┐
│                  WHAT CONTEXT ENGINEERING GUARANTEES     │
│                                                          │
│  ┌─────────────┐    ┌────────────────┐    ┌───────────┐  │
│  │  Raw Data   │───▶│Context Engineer│───▶│   LLM     │  │
│  │  Sources    │    │                │    │  Context  │  │
│  │             │    │  • Relevant    │    │  Window   │  │
│  │  Loki       │    │  • Filtered    │    │           │  │
│  │  Prometheus │    │  • Compressed  │    │  Fits:    │  │
│  │  Kafka      │    │  • Formatted   │    │  < 128k   │  │
│  │  MongoDB    │    │  • Sourced     │    │  tokens   │  │
│  └─────────────┘    └────────────────┘    └───────────┘  │
└──────────────────────────────────────────────────────────┘
```

---

## 6. Step-by-Step Walkthrough

**Step 1** — Query arrives: *"Why has MarketID ABC stopped processing trades since 10:30?"*

**Step 2** — Context engineering layer extracts key parameters:
- Entity: MarketID ABC
- Time anchor: 10:30
- Query type: Outage / Root Cause

**Step 3** — Determines which sources are relevant for an outage query: Loki (error logs), Prometheus (service health), Kafka (consumer lag).

**Step 4** — Fetches from each source with precise filters. Does **not** fetch everything.

**Step 5** — Scores and ranks retrieved items by relevance (time proximity, severity, entity match).

**Step 6** — Trims to fit within a pre-allocated token budget (e.g., 15k tokens for tool results).

**Step 7** — Formats each source with clear headers and labels.

**Step 8** — Assembles final context: system prompt + formatted evidence + user question.

---

## 7. Enterprise Design

```
┌──────────────────────────────────────────────────────────┐
│              CONTEXT ENGINEERING ARCHITECTURE            │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │              CONTEXT BUDGET MANAGER             │    │
│  │                                                 │    │
│  │  Total Window: 128k tokens                      │    │
│  │  System Prompt: 2k (reserved)                   │    │
│  │  Output Reserve: 4k (reserved)                  │    │
│  │  Available for context: 122k                    │    │
│  │                                                 │    │
│  │  Allocation per source type:                    │    │
│  │    Logs: max 40k                                │    │
│  │    Metrics: max 10k                             │    │
│  │    DB Records: max 20k                          │    │
│  │    Previous findings: max 10k                   │    │
│  └─────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

---

## 8. Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| No context budget | One large tool result fills window; other sources get cut |
| Injecting data without source labels | LLM cannot distinguish which claim came from which system |
| Unsorted context | LLM gives higher attention to text at the beginning and end; middle gets lost |
| No summarization step for large results | 50k token Loki dump → truncation → missed root cause |
| Static context assembly (same for all query types) | Latency queries need different evidence than outage queries |

---

## 9. Summary

**Key Ideas:**
- Context engineering is evidence selection and preparation, not just prompt writing
- What you leave out is as important as what you include
- Format and structure of injected context affects LLM reasoning quality
- Token budgeting per source prevents one noisy source from crowding out others

**Mental Model:**
> Context engineering is what a good analyst does before bringing a problem to a senior engineer. They don't dump all the logs on the table. They prepare a summary: "Here's the timeline, here are the three most relevant error messages, here's the metric that confirms the impact."

---

# Chapter 6: Structured Outputs

## 1. Definition

**Structured outputs** are LLM responses that conform to a predefined schema — JSON, XML, or another format — rather than free-form prose.

Instead of:
> *"It looks like the trade processor for MarketID ABC may have experienced a Kafka connection issue around 10:30 based on the logs..."*

You get:
```json
{
  "root_cause": "Kafka broker connection timeout",
  "confidence": "HIGH",
  "first_error_time": "10:29:58",
  "affected_service": "trade-processor-abc",
  "evidence": [
    {"source": "Loki", "line": "FATAL Connection refused: kafka-broker-1:9092"},
    {"source": "Prometheus", "metric": "kafka_consumer_lag", "value": 45000}
  ],
  "next_step": "Check Kafka broker health via kafka-broker-1:9092"
}
```

The second form is *parseable by code*. The first is not.

---

## 2. Why It Exists

Agents are software systems. They pass data between components. If one component (the LLM) outputs prose, the next component (the orchestrator, the UI, the database writer) cannot reliably parse it.

Structured outputs make LLMs composable with the rest of your software stack.

Without them: you'd need fragile regex/string-parsing logic to extract findings from prose — and a single sentence variation breaks your pipeline.

---

## 3. Real-World Analogy

```
┌─────────────────────────────────────────────────────────┐
│                LAB TEST RESULTS                         │
│                                                         │
│  UNSTRUCTURED: "Your cholesterol is a bit high, and    │
│  your blood pressure readings suggest some concern,     │
│  though it's not at a critical level yet..."            │
│                                                         │
│  STRUCTURED:                                            │
│  Cholesterol:      215 mg/dL   [Normal: <200]  ⚠ HIGH  │
│  Blood Pressure:   138/88      [Normal: <120]  ⚠ HIGH  │
│  Blood Glucose:    92 mg/dL    [Normal: 70-99] ✓ NORMAL│
│                                                         │
│  Structured results: a nurse can act without reading    │
│  a paragraph. Your orchestrator can act on structured   │
│  LLM output without parsing prose.                      │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Observability Example

Your RCA agent returns this structured output after investigation:

```json
{
  "investigation_id": "INV-2024-0115-001",
  "market_id": "ABC",
  "outage_start": "2024-01-15T10:29:58Z",
  "root_cause": {
    "description": "Kafka broker kafka-broker-1 became unavailable",
    "category": "INFRASTRUCTURE",
    "confidence": "HIGH"
  },
  "symptoms": [
    "Kafka consumer lag for ABC-processor grew from 0 to 45,000",
    "trade-processor-abc service reported connection refused",
    "Zero trades processed after 10:30:01"
  ],
  "evidence": [
    {
      "source": "Loki",
      "timestamp": "2024-01-15T10:29:58Z",
      "content": "FATAL Connection refused: kafka-broker-1:9092"
    },
    {
      "source": "Prometheus",
      "metric": "kafka_consumer_lag",
      "value": 45000,
      "observed_at": "2024-01-15T10:31:00Z"
    }
  ],
  "recommended_actions": [
    "SSH to kafka-broker-1 and check broker status",
    "Review Kafka broker logs for the crash cause",
    "Consider failing over to kafka-broker-2"
  ],
  "escalate": false
}
```

Every field is machine-readable. Your orchestrator can: write this to MongoDB, trigger a PagerDuty alert if confidence=HIGH, display it in a Grafana panel, or feed it to a remediation agent.

---

## 5. Visual Workflow

### From Prose to Structured: The Pipeline

```
LLM Raw Output (prose)
"The trade processor for MarketID ABC stopped at 10:30
 due to what appears to be a Kafka connection failure
 based on the error logs..."
         │
         ▼
   OUTPUT PARSER
   (JSON extraction,
    schema validation)
         │
    ┌────┴────┐
    │ Valid?  │
    │ JSON?   │
    └────┬────┘
    YES  │  NO
         │   └──▶ Retry with rephrased prompt
         ▼         "Respond ONLY in JSON format..."
Validated Structured Object
         │
         ├──▶  Write to MongoDB (investigation record)
         ├──▶  Trigger PagerDuty if escalate=true
         ├──▶  Display in RCA UI
         └──▶  Feed to Remediation Agent
```

### Enforcing Structure: Three Approaches

```
┌──────────────────────────────────────────────────────────────┐
│           METHODS TO ENFORCE STRUCTURED OUTPUT               │
│                                                              │
│  1. PROMPT INSTRUCTION                                       │
│     "Respond ONLY in valid JSON. No other text.             │
│      Schema: { root_cause: string, confidence: string ... }" │
│     Reliability: Medium (LLM may still add prose)           │
│                                                              │
│  2. JSON MODE (API Feature)                                  │
│     Force the model at API level to only produce JSON        │
│     Reliability: High (enforced by API)                      │
│                                                              │
│  3. FUNCTION CALLING / TOOL USE                              │
│     Define the schema as a function signature               │
│     LLM "calls" the function with structured arguments       │
│     Reliability: Very High (schema-enforced)                 │
│     → This is Chapter 7                                      │
└──────────────────────────────────────────────────────────────┘
```

### Schema Design for RCA Output

```
RCA_RESULT Schema
├── investigation_id  (string, required)
├── market_id         (string, required)
├── outage_start      (ISO8601 datetime, required)
├── root_cause        (object, required)
│   ├── description   (string)
│   ├── category      (enum: INFRASTRUCTURE | APPLICATION | DATA | NETWORK)
│   └── confidence    (enum: HIGH | MEDIUM | LOW)
├── symptoms          (array of strings)
├── evidence          (array of objects)
│   ├── source        (enum: Loki | Prometheus | Kafka | MongoDB)
│   ├── timestamp     (ISO8601, optional)
│   └── content       (string)
├── recommended_actions (array of strings)
└── escalate          (boolean)
```

---

## 6. Step-by-Step Walkthrough

**Step 1** — System prompt instructs: "Output only JSON matching this schema: {...}"

**Step 2** — LLM generates its analysis and formats it as JSON (ideally using JSON mode or function calling).

**Step 3** — Your orchestrator receives the raw LLM response string.

**Step 4** — A parser extracts the JSON block (stripping any preamble the LLM added).

**Step 5** — A schema validator (e.g., Pydantic, JSON Schema) validates the output.

**Step 6** — If invalid: retry the LLM with an error message: "Your previous output was invalid JSON. Error: [details]. Please retry."

**Step 7** — If valid: route the structured object to downstream systems.

---

## 7. Enterprise Design

| Pattern | Description |
|---------|-------------|
| **Schema-first design** | Define the output schema before writing the prompt |
| **Graceful retry** | On parse failure, retry with validation error as context |
| **Partial extraction** | If LLM outputs partial JSON, extract what's valid |
| **Schema versioning** | As RCA requirements evolve, version your output schemas |
| **Downstream contract** | Treat the structured output as an API contract with consumers |

---

## 8. Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| No schema validation | Invalid JSON silently breaks downstream pipeline |
| Schema too rigid | LLM cannot express genuine uncertainty; produces forced output |
| No retry logic | One malformed response halts the investigation |
| Deeply nested schemas | LLM loses track of nesting depth; produces malformed output |

---

## 9. Summary

**Key Ideas:**
- Structured outputs make LLMs composable with software systems
- JSON mode and function calling are more reliable than prompt instructions alone
- Always validate output against a schema before trusting it
- Design the schema to match what downstream systems need, not what's easy to generate

**Mental Model:**
> A structured output contract is like an API specification. The LLM is a service. Define its response schema upfront. Validate every response. Retry on failure. Never trust raw prose in a production pipeline.

---

# Chapter 7: Function Calling

## 1. Definition

**Function calling** (also called tool use) is a mechanism where the LLM, instead of generating a prose answer, generates a *structured request to call a specific function* with specific parameters.

The LLM does not execute the function. It generates the call specification. Your code executes it.

Example — LLM generates:
```json
{
  "function": "query_loki_logs",
  "arguments": {
    "service": "trade-processor",
    "market_id": "ABC",
    "severity": "ERROR",
    "start_time": "2024-01-15T10:15:00Z",
    "end_time": "2024-01-15T10:45:00Z"
  }
}
```

Your code sees this, calls the actual Loki API, and returns the results to the LLM.

---

## 2. Why It Exists

Before function calling, getting an LLM to interact with external systems required:
- Prompting the LLM to output a specific text format representing an action
- Writing fragile parsers to extract the action from prose
- Handling all the edge cases where the LLM deviated from the format

Function calling solves this by making tool invocation a first-class feature of the LLM API. The model is trained to recognize when a function should be called and to generate a structured, schema-valid call specification.

This is what transforms an LLM from a text generator into an **agent**.

---

## 3. Real-World Analogy

```
┌─────────────────────────────────────────────────────────┐
│              A MANAGER DELEGATING TASKS                 │
│                                                         │
│  Manager (LLM) thinks: "I need to know the Kafka lag   │
│  for MarketID ABC."                                     │
│                                                         │
│  Manager doesn't pull up Prometheus themselves.         │
│  They write a task ticket:                              │
│                                                         │
│  TASK:   Query Kafka consumer lag                       │
│  PARAMS: market=ABC, time_range=10:00-11:00             │
│  ASSIGN: Prometheus analyst                             │
│                                                         │
│  The analyst (your code) executes. Reports back.        │
│  Manager uses the result in their analysis.             │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Observability Example

You define these functions for your RCA agent:

```
AVAILABLE FUNCTIONS:
┌──────────────────────────────────────────────────────────┐
│  query_loki_logs(service, market_id, severity,           │
│                  start_time, end_time, limit)            │
│  → Returns: array of log lines                           │
│                                                          │
│  query_prometheus_metrics(metric_name, labels,           │
│                           start_time, end_time, step)    │
│  → Returns: time series data points                      │
│                                                          │
│  get_kafka_consumer_group_state(group_id)                │
│  → Returns: lag, offset, status per partition            │
│                                                          │
│  get_trade_records(market_id, start_time, status)        │
│  → Returns: array of trade records from MongoDB          │
│                                                          │
│  get_service_health(service_name)                        │
│  → Returns: Grafana health status for the service        │
└──────────────────────────────────────────────────────────┘
```

When the user asks *"Why has MarketID ABC stopped processing trades since 10:30?"*, the LLM decides to call `query_loki_logs` first. Then `query_prometheus_metrics`. Then reasons on the combined results.

---

## 5. Visual Workflow

### Function Calling Sequence

```
User: "Why has MarketID ABC stopped processing trades since 10:30?"
                          │
                          ▼
              ┌───────────────────────┐
              │         LLM           │
              │  Sees: question +     │
              │  available functions  │
              │  Decides: I need logs │
              └───────────┬───────────┘
                          │ Generates function call:
                          ▼
              ┌───────────────────────┐
              │  {                    │
              │    "function":        │
              │    "query_loki_logs", │
              │    "arguments": {     │
              │      "service":       │
              │      "trade-processor"│
              │      "market_id":"ABC"│
              │      "severity":"ERR" │
              │      "start":"10:15Z" │
              │    }                  │
              │  }                    │
              └───────────┬───────────┘
                          │
                          ▼ Your code executes this
              ┌───────────────────────┐
              │      Loki API         │
              │  Returns 147 log lines│
              └───────────┬───────────┘
                          │ Results returned to LLM context
                          ▼
              ┌───────────────────────┐
              │         LLM           │
              │  Sees: log results    │
              │  Decides: also need   │
              │  Prometheus metrics   │
              └───────────┬───────────┘
                          │ Generates 2nd function call
                          ▼
              ┌───────────────────────┐
              │   Prometheus API      │
              │  Returns lag metrics  │
              └───────────┬───────────┘
                          │ Results returned to LLM context
                          ▼
              ┌───────────────────────┐
              │         LLM           │
              │  Has: logs + metrics  │
              │  Generates: RCA JSON  │
              └───────────────────────┘
```

### Function Calling vs. Prose Parsing

```
BEFORE FUNCTION CALLING:
LLM Output:  "I need to check the Loki logs for service trade-processor..."
Code:        Parse the prose → extract service name → hope format matches
Risk:        LLM says "trade processor" instead of "trade-processor" → regex fails

AFTER FUNCTION CALLING:
LLM Output:  { "function": "query_loki_logs", "arguments": { "service": "trade-processor" } }
Code:        Directly call the function with validated arguments
Risk:        Near zero — schema is enforced by the API
```

### Multi-Step Function Call Chain for RCA

```
Question: "Why has MarketID ABC stopped processing trades since 10:30?"

Step 1: LLM calls query_loki_logs(market_id="ABC", severity="ERROR", ...)
         → 147 error lines returned

Step 2: LLM calls get_kafka_consumer_group_state(group_id="ABC-processor")
         → lag=45000, status=LAGGING

Step 3: LLM calls query_prometheus_metrics(metric="trade_processor_up", labels={market="ABC"})
         → value=0 since 10:30:01

Step 4: LLM has sufficient evidence → generates structured RCA JSON
         No more function calls needed.
```

---

## 6. Step-by-Step Walkthrough

**Step 1** — You define function schemas (name, description, parameter types) and pass them to the LLM API alongside the user's question.

**Step 2** — The LLM reads the question and the available functions. It decides: *"I need log data to answer this. I will call `query_loki_logs`."*

**Step 3** — The LLM outputs a function call specification (JSON).

**Step 4** — Your orchestrator detects that the output is a function call (not a final answer).

**Step 5** — Your code executes the actual API call to Loki.

**Step 6** — The results are appended to the conversation as a "function result" message.

**Step 7** — The LLM sees the results. It may call another function or generate a final answer.

**Step 8** — This loop continues until the LLM generates a final answer (no more function calls).

---

## 7. Enterprise Design

| Design Decision | Options | Recommendation |
|-----------------|---------|----------------|
| Parallel vs. sequential calls | LLM can call one function at a time or multiple in parallel | Parallel where possible (reduces latency) |
| Max function call depth | Unbounded or capped | Always cap (e.g., max 10 iterations) to prevent infinite loops |
| Function result size limit | Unlimited or capped | Cap at 10k tokens per result; summarize if larger |
| Error handling | Return errors to LLM or abort | Return errors; let LLM adapt its strategy |

---

## 8. Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| No max iteration cap | Agent enters infinite loop calling the same function repeatedly |
| Too many functions registered | LLM gets confused; selects wrong function |
| Vague function descriptions | LLM calls the wrong function or uses wrong parameters |
| No error handling for function results | One API timeout breaks the entire investigation |
| Trusting function arguments without validation | LLM generates `start_time="ten thirty"` instead of ISO8601 |

---

## 9. Summary

**Key Ideas:**
- Function calling is the mechanism that transforms LLMs into agents
- The LLM decides *what* to call and *with what arguments*; your code executes it
- Schema-enforced function definitions are more reliable than prose-based action extraction
- Always cap iterations and validate arguments before execution

**Mental Model:**
> The LLM is a manager who writes precise work orders. Function calling is the work order format. Your code is the team that executes the work orders. The manager reviews the results and writes the next work order — until the job is done.

---

# Chapter 8: Tool Calling

## 1. Definition

**Tool calling** is the broader pattern of an agent using external capabilities — APIs, databases, code executors, search engines — to gather information or take actions it cannot do by itself.

Function calling (Chapter 7) is the *mechanism* by which an LLM specifies a tool call. Tool calling is the *architectural pattern* of having a catalog of capabilities and an agent that selects from them.

Think of it this way:
- **Function calling** = the syntax (how the LLM requests a tool)
- **Tool calling** = the system (what tools exist, how they're registered, how results are returned)

---

## 2. Why It Exists

An LLM alone can:
- Reason about problems
- Generate text
- Classify and summarize

An LLM with tools can:
- Query your live Loki logs
- Read real Prometheus metrics
- Look up MarketID ABC in MongoDB
- Check Kafka consumer group state
- Create a PagerDuty incident

The tool calling pattern is what makes LLMs useful for real operational tasks rather than just answering general knowledge questions.

---

## 3. Real-World Analogy

```
┌─────────────────────────────────────────────────────────┐
│                  AN IT SUPPORT ENGINEER                 │
│                                                         │
│  Without tools:  Can think about the problem,           │
│                  give advice based on experience        │
│                                                         │
│  With tools:     - Opens monitoring dashboard           │
│                  - Runs log grep command                 │
│                  - Queries the CMDB                     │
│                  - Restarts a service via SSH           │
│                  - Opens a JIRA ticket                  │
│                                                         │
│  The engineer = LLM (reasoning)                         │
│  The tools    = actual access to systems                │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Observability Example

Your RCA platform's tool catalog:

```
┌──────────────────────────────────────────────────────────────┐
│                     RCA AGENT TOOL CATALOG                   │
│                                                              │
│  OBSERVABILITY TOOLS                                         │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  loki_log_query        Query log data with LogQL        │  │
│  │  prometheus_metric     Query metrics with PromQL        │  │
│  │  grafana_dashboard     Get panel data from dashboard    │  │
│  │  tempo_trace_search    Search distributed traces        │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  DATA TOOLS                                                  │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  mongodb_trade_query   Query trade records              │  │
│  │  snowflake_analytics   Query historical analytics       │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  INFRASTRUCTURE TOOLS                                        │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  kafka_consumer_state  Get consumer group lag/state     │  │
│  │  kafka_topic_metadata  Get topic partition info         │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ACTION TOOLS                                                │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  create_pagerduty_incident  Escalate to on-call         │  │
│  │  post_slack_alert           Notify trading ops team     │  │
│  │  create_jira_ticket         Log incident for follow-up  │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

---

## 5. Visual Workflow

### Tool Calling Architecture

```
                         USER QUERY
                              │
                              ▼
                    ┌─────────────────┐
                    │      LLM        │
                    │  (Reasoning)    │
                    └────────┬────────┘
                             │ Tool call request
                             ▼
                    ┌─────────────────┐
                    │  TOOL EXECUTOR  │
                    │  (Orchestrator) │
                    └────────┬────────┘
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
          ▼                  ▼                  ▼
   ┌────────────┐    ┌─────────────┐    ┌──────────────┐
   │    Loki    │    │ Prometheus  │    │    Kafka     │
   │  Log Query │    │  Metrics   │    │   Consumer   │
   └──────┬─────┘    └──────┬──────┘    └──────┬───────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  TOOL RESULTS   │
                    │  (aggregated)   │
                    └────────┬────────┘
                             │ Injected into context
                             ▼
                    ┌─────────────────┐
                    │      LLM        │
                    │  (Reasoning     │
                    │   on results)   │
                    └────────┬────────┘
                             │
                             ▼
                       FINAL RCA OUTPUT
```

### Tool Selection Decision Tree

```
                User Query: "MarketID ABC not processing"
                              │
                              ▼
                  Is this an outage or latency issue?
                      │                   │
                    OUTAGE              LATENCY
                      │                   │
                      ▼                   ▼
           ┌─────────────────┐   ┌─────────────────────┐
           │ Query Loki:     │   │ Query Prometheus:    │
           │ ERROR/FATAL logs│   │ p95/p99 latency      │
           └───────┬─────────┘   └──────────┬──────────┘
                   │                        │
                   ▼                        ▼
           ┌─────────────────┐   ┌─────────────────────┐
           │ Query Kafka:    │   │ Query Tempo:         │
           │ Consumer lag    │   │ Trace details        │
           └─────────────────┘   └─────────────────────┘
```

---

## 6. Step-by-Step Walkthrough

**Step 1** — LLM receives: user question + list of available tools with descriptions.

**Step 2** — LLM reasons: "This is a trade outage. I need error logs and infrastructure state."

**Step 3** — LLM generates tool call: `loki_log_query` with parameters.

**Step 4** — Orchestrator validates the tool call (tool exists? parameters valid?).

**Step 5** — Orchestrator executes the tool (calls Loki API).

**Step 6** — Result is returned to LLM context.

**Step 7** — LLM generates next tool call: `kafka_consumer_state`.

**Step 8** — Process repeats until LLM has sufficient evidence.

**Step 9** — LLM generates final structured RCA output. No tool call — this is the signal to stop.

**Step 10** — If escalation warranted: orchestrator calls `create_pagerduty_incident` tool.

---

## 7. Enterprise Design

| Concern | Design | Applied to RCA |
|---------|--------|----------------|
| **Tool timeouts** | Each tool call has max timeout | Loki query: 10s, Kafka: 5s, MongoDB: 15s |
| **Tool error handling** | Return error to LLM with details | LLM adapts: tries different query |
| **Parallel tool calls** | Execute non-dependent calls simultaneously | Loki + Prometheus can be called in parallel |
| **Tool result caching** | Cache results within session | Don't re-query same Loki range twice |
| **Rate limiting** | Respect API limits | Prometheus: max 10 calls/min |

---

## 8. Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| Action tools with no human approval | Agent creates PagerDuty incidents for false positives |
| No tool timeout handling | One slow MongoDB query blocks entire investigation |
| Tools that return too much data | Fills context window; later tools get squeezed out |
| No tool result validation | Malformed API response crashes the agent |

---

## 9. Summary

**Key Ideas:**
- Tools are the "arms" of the agent — they enable action and data retrieval
- Function calling is the mechanism; tool calling is the architectural pattern
- Tool catalogs should be designed around the agent's reasoning needs
- Action tools (write, create, modify) need stricter controls than read tools

**Mental Model:**
> Tool calling turns the LLM from a librarian (who only knows what they've read) into an investigator (who can go pull new evidence from any system they have access to).

---

# Chapter 9: MCP (Model Context Protocol)

## 1. Definition

**MCP (Model Context Protocol)** is an open standard that defines how AI agents communicate with external tools and data sources in a uniform way.

Before MCP: every tool integration was custom. To connect your agent to Loki, you wrote a Loki-specific integration. To connect to Prometheus, another custom integration. 20 tools = 20 custom integrations, each with its own quirks.

After MCP: tools expose a standard MCP interface. Your agent speaks MCP. Any MCP-compatible tool connects without custom code.

MCP is to AI agents what USB is to devices: a universal connector.

---

## 2. Why It Exists

The AI tooling ecosystem fragmented quickly. Every framework invented its own tool integration format. Tools written for LangChain didn't work with OpenAI agents. Tools for Claude didn't work with Gemini.

MCP (published by Anthropic in late 2024) addresses this by defining:
- How tools describe themselves (their schema)
- How agents request tool execution
- How results are returned
- How authentication is handled

It decouples the *tool* from the *agent framework*.

---

## 3. Real-World Analogy

```
┌─────────────────────────────────────────────────────────┐
│               ELECTRICAL STANDARDS                      │
│                                                         │
│  Before standard plugs: every device had a custom      │
│  connector. Buying a lamp from a different country      │
│  required an adapter (or rewiring).                    │
│                                                         │
│  After standard: any device (tool) with a standard     │
│  plug works in any socket (agent framework).           │
│                                                         │
│  MCP = electrical standard for AI tool integration     │
│  MCP Server = the tool (lamp)                          │
│  MCP Client = the agent (socket)                       │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Observability Example

Without MCP, your RCA platform looks like this:

```
BEFORE MCP: Custom integrations everywhere

Agent ──▶ Custom Loki Connector ──▶ Loki
Agent ──▶ Custom Prometheus Connector ──▶ Prometheus
Agent ──▶ Custom Kafka Connector ──▶ Kafka
Agent ──▶ Custom MongoDB Connector ──▶ MongoDB

Problem: Change agent framework → rebuild all 4 connectors
         Add new data source → write new custom connector
```

With MCP:

```
AFTER MCP: Standard protocol

Agent (MCP Client)
   │
   │ Standard MCP Protocol
   ├──▶ Loki MCP Server ──▶ Loki
   ├──▶ Prometheus MCP Server ──▶ Prometheus
   ├──▶ Kafka MCP Server ──▶ Kafka
   └──▶ MongoDB MCP Server ──▶ MongoDB

Benefit: Change agent framework → zero changes to tool servers
         Add new data source → expose it as MCP server → done
```

---

## 5. Visual Workflow

### MCP Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                      MCP ECOSYSTEM                           │
│                                                              │
│  ┌─────────────────────────────────────┐                     │
│  │           MCP CLIENT (Agent)         │                     │
│  │                                     │                     │
│  │  LangGraph / Claude / OpenAI Agent  │                     │
│  │  Speaks MCP protocol                │                     │
│  └──────────────────┬──────────────────┘                     │
│                     │  MCP Protocol (JSON-RPC over HTTP/SSE) │
│     ┌───────────────┼───────────────┐                        │
│     │               │               │                        │
│     ▼               ▼               ▼                        │
│  ┌──────┐       ┌──────┐       ┌──────┐                      │
│  │ Loki │       │Prom  │       │Kafka │                      │
│  │ MCP  │       │ MCP  │       │ MCP  │                      │
│  │Server│       │Server│       │Server│                      │
│  └──┬───┘       └──┬───┘       └──┬───┘                      │
│     │              │              │                           │
│     ▼              ▼              ▼                           │
│   Loki API     Prom API      Kafka API                        │
└──────────────────────────────────────────────────────────────┘
```

### MCP Message Flow

```
Agent                    MCP Server (Loki)
  │                           │
  │  1. list_tools()          │
  │ ─────────────────────────▶│
  │                           │
  │  2. Tools: [query_logs,   │
  │     get_labels, ...]      │
  │ ◀─────────────────────────│
  │                           │
  │  3. call_tool(            │
  │     "query_logs",         │
  │     {service: "trade-    │
  │      processor",         │
  │      severity: "ERROR"}) │
  │ ─────────────────────────▶│
  │                           │
  │                           │ Calls Loki API
  │                           │ with LogQL query
  │                           │
  │  4. Result:               │
  │     [147 log lines]       │
  │ ◀─────────────────────────│
  │                           │
```

### MCP vs. Direct Integration

```
┌──────────────────────────────────────────────────────────────┐
│                  COMPARISON TABLE                            │
├──────────────────┬───────────────────┬───────────────────────┤
│ Dimension        │ Direct Integration│ MCP                   │
├──────────────────┼───────────────────┼───────────────────────┤
│ Framework lock-in│ High              │ None                  │
│ Setup effort     │ Custom per tool   │ One-time per tool     │
│ Tool portability │ Not portable      │ Any MCP client works  │
│ Discovery        │ Manual            │ list_tools() API      │
│ Auth handling    │ Custom            │ Standardized          │
│ Community tools  │ None              │ Growing ecosystem     │
└──────────────────┴───────────────────┴───────────────────────┘
```

---

## 6. Step-by-Step Walkthrough

**Step 1** — Your RCA agent starts. It calls `list_tools()` on each registered MCP server.

**Step 2** — Loki MCP Server returns: `[query_logs, get_label_values, get_log_streams]`

**Step 3** — Prometheus MCP Server returns: `[query_instant, query_range, list_metrics]`

**Step 4** — The agent now knows all available tools and their schemas.

**Step 5** — User asks about MarketID ABC. Agent generates tool call using the standard MCP format.

**Step 6** — MCP client sends the call to the correct MCP server.

**Step 7** — MCP server calls the real Loki API, formats the result, returns it via MCP protocol.

**Step 8** — Agent receives results in a uniform format, regardless of which tool was called.

---

## 7. Enterprise Design

| MCP Deployment Pattern | Description | When to Use |
|------------------------|-------------|-------------|
| **Local MCP servers** | MCP server runs as sidecar to agent | Low-latency, single-datacenter |
| **Remote MCP servers** | MCP server exposed as HTTP endpoint | Shared tools across multiple agents |
| **MCP gateway** | Single entry point that routes to multiple MCP servers | Centralized auth and rate limiting |
| **MCP discovery service** | Registry of all available MCP servers | Large enterprises with many tools |

---

## 8. Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| Building custom integrations instead of MCP servers | Framework migration requires rebuilding everything |
| No MCP server authentication | Any process on the network can invoke your tools |
| MCP servers with side effects and no approval gate | Agent accidentally triggers production changes |
| Ignoring MCP server errors | Silent failures; agent reasons on missing data |

---

## 9. Summary

**Key Ideas:**
- MCP is a universal standard for agent-tool communication
- One MCP server, any framework — portability is the core value
- MCP provides tool discovery, structured calls, and standard result formats
- In production, always add authentication and rate limiting to MCP servers

**Mental Model:**
> MCP is the USB standard of AI tooling. Once a tool speaks MCP, any agent can use it — regardless of which framework the agent is built on. You invest once in the tool server; every agent benefits.

---

# Chapter 10: Intent

## 1. Definition

**Intent** is the underlying goal or purpose behind a user's query — what they actually want to accomplish, independent of the specific words they used.

When a user says:
> *"Why has MarketID ABC stopped processing trades since 10:30?"*

The **intent** is: *Investigate and diagnose a trade processing outage for a specific market.*

This is different from:
- The literal question (what words were used)
- The entities mentioned (MarketID ABC, 10:30)
- The tone (urgent, curious, frustrated)

Intent is the *what do they need* layer, above the *what did they say* layer.

---

## 2. Why It Exists

Users don't talk to systems like APIs. They use natural language, which is ambiguous:

| What user says | Possible intents |
|----------------|-----------------|
| "MarketID ABC is down" | Investigate outage? Get status? Alert someone? |
| "10:30 trades" | What happened at 10:30? How many trades processed? Which trades? |
| "Something's wrong with ABC" | Outage? Data quality? Latency? Config change? |

Without intent classification, an agent doesn't know *which type of investigation to run*. It might query logs when the user wanted a status summary. It might generate a report when the user wanted an alert sent.

Intent is the **routing decision** that determines everything the agent does next.

---

## 3. Real-World Analogy

```
┌─────────────────────────────────────────────────────────┐
│                    CALL CENTER ROUTING                  │
│                                                         │
│  Customer: "I have a problem with my account"          │
│                                                         │
│  Without intent routing:  Random department picks up   │
│                                                         │
│  With intent routing:                                   │
│  System detects: billing complaint? password reset?    │
│  technical issue? → Routes to correct specialist       │
│                                                         │
│  Your agent: detects if it's an outage investigation,  │
│  a performance query, a data quality check,            │
│  or a general how-to question.                         │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Observability Example

For your RCA platform, define a clear intent taxonomy:

```
┌──────────────────────────────────────────────────────────────┐
│                   INTENT TAXONOMY: RCA PLATFORM              │
│                                                              │
│  OUTAGE_INVESTIGATION                                        │
│  "Why is X down?" / "X stopped working" / "X not processing" │
│  → Triggers: Loki error logs + Kafka lag + service health    │
│                                                              │
│  LATENCY_INVESTIGATION                                       │
│  "X is slow" / "X taking too long" / "latency spike"        │
│  → Triggers: Prometheus p95/p99 + Tempo traces               │
│                                                              │
│  DATA_QUALITY_CHECK                                          │
│  "X data looks wrong" / "missing trades" / "bad values"      │
│  → Triggers: MongoDB trade records + Snowflake analytics     │
│                                                              │
│  STATUS_CHECK                                                │
│  "Is X healthy?" / "What's the current state of X?"         │
│  → Triggers: Grafana dashboard + service health              │
│                                                              │
│  TREND_ANALYSIS                                              │
│  "How has X been performing?" / "Show me X over the week"    │
│  → Triggers: Prometheus long-range + Snowflake               │
│                                                              │
│  GENERAL_QUERY                                               │
│  "What is X?" / "How does X work?" / "What does error Y mean"│
│  → Triggers: Knowledge base / documentation search           │
└──────────────────────────────────────────────────────────────┘
```

*"Why has MarketID ABC stopped processing trades since 10:30?"*
→ Intent: **OUTAGE_INVESTIGATION**

---

## 5. Visual Workflow

### Intent Detection Pipeline

```
User Query
    │
    ▼
┌──────────────────────────────────────────────────┐
│                INTENT CLASSIFIER                  │
│                                                  │
│  Method: LLM prompt / fine-tuned classifier      │
│  Input:  Raw user query                          │
│  Output: Intent label + confidence score         │
└──────────────────────┬───────────────────────────┘
                       │
         ┌─────────────┼─────────────────────────┐
         │             │             │             │
         ▼             ▼             ▼             ▼
   OUTAGE_INV   LATENCY_INV   DATA_QUALITY   STATUS_CHECK
       │              │             │              │
       ▼              ▼             ▼              ▼
  Loki+Kafka   Prometheus    MongoDB+Snow    Grafana
  deep dive    +Tempo        flake checks   summary
```

### Intent Drives Everything Downstream

```
                            INTENT = OUTAGE_INVESTIGATION
                                         │
              ┌──────────────────────────┼──────────────────────────┐
              │                          │                          │
              ▼                          ▼                          ▼
       TOOL SELECTION            CONTEXT STRATEGY             OUTPUT FORMAT
       ─────────────            ────────────────             ─────────────
       Loki ERROR logs          15min window               Root Cause JSON
       Kafka consumer lag       around 10:30               with confidence
       Service health           high-severity first        and next steps
```

### Intent vs. Other Concepts

```
User Query: "Why has MarketID ABC stopped processing trades since 10:30?"
     │
     ├──▶ INTENT:   OUTAGE_INVESTIGATION   (what they want to do)
     │
     ├──▶ ENTITIES: MarketID=ABC, Time=10:30   (what they're talking about)
     │               [covered in Chapter 11]
     │
     ├──▶ TONE:     Urgent, factual              (how they feel)
     │
     └──▶ CONTEXT:  Trading hours, production    (implied background)
```

---

## 6. Step-by-Step Walkthrough

**Step 1** — User submits query.

**Step 2** — Intent classifier runs. Two approaches:
- LLM-based: "Classify this query into one of: [OUTAGE_INVESTIGATION, LATENCY_INVESTIGATION, DATA_QUALITY_CHECK, STATUS_CHECK, TREND_ANALYSIS, GENERAL_QUERY]"
- Fine-tuned classifier: faster, cheaper, but requires training data

**Step 3** — Classification result: `{ "intent": "OUTAGE_INVESTIGATION", "confidence": 0.97 }`

**Step 4** — If confidence < 0.7: ask the user for clarification. "Are you reporting a service outage or checking trade data completeness?"

**Step 5** — Intent is stored in the agent's state (Chapter 19).

**Step 6** — Planner (Chapter 14) uses the intent to select the appropriate investigation strategy and tools.

**Step 7** — Every downstream decision — which tools to call, what context to retrieve, what output format to generate — is informed by the intent.

---

## 7. Enterprise Design

| Pattern | Description | Trade-off |
|---------|-------------|-----------|
| **LLM-based classification** | Use the main LLM to classify intent | Flexible, no training data needed; slower, costlier |
| **Fine-tuned classifier** | Train a small model on historical queries | Fast, cheap; needs labeled training data |
| **Rule-based pre-filter** | Regex/keyword matching for obvious cases | Very fast; brittle for edge cases |
| **Hybrid** | Rules first, LLM for ambiguous cases | Best performance; more complex |

For a financial trading platform with defined query types, a fine-tuned classifier on your historical trader queries is the right long-term investment.

---

## 8. Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| No intent classification | Agent runs same investigation for every query type |
| Too many intent categories | Categories overlap; classifier becomes unreliable |
| Low confidence threshold | Too many ambiguous queries reach clarification stage |
| Not handling multi-intent queries | "Is ABC down, and what was the trade volume yesterday?" → two intents |
| Hardcoding intent-to-tool mapping | New query types require code changes |

---

## 9. Summary

**Key Ideas:**
- Intent is the purpose behind the query, not the words used
- Intent classification is the first routing decision in an agent system
- Everything downstream — tools, context strategy, output format — follows from intent
- Confidence thresholds and clarification flows are essential for production

**Mental Model:**
> Intent is the triage nurse's decision at the hospital entrance. Before any doctor examines the patient, the triage nurse classifies the severity and type of problem. That classification determines which ward, which specialist, and how urgently. Your intent classifier is that triage nurse — it determines everything that happens next.

**Interview Questions:**
1. What is the difference between intent and entity?
2. How would you handle a query that contains two different intents?
3. What confidence threshold would you use before asking for clarification?

**Self-Check Questions:**
- How many distinct intent categories does your platform need?
- What happens to a query classified as OUTAGE_INVESTIGATION that actually turned out to be a DATA_QUALITY issue?

**Practical Exercise:**
Collect 50 real queries from your trading operations team. Manually label each with an intent. Then ask yourself: are there categories you hadn't considered? Are any categories ambiguous? This exercise defines your intent taxonomy.

---

*[End of Part 1 — Chapters 1–10]*
