---
layout: post
comments: false
title:  "Memory Architecture"
date:   2026-05-30 10:00:00
---
# Memory Architecture, Working Memory Management & Summarization in Agentic AI Systems

_AI generated article, crafted using Claude as per my own implementation experience_

## 1. Memory Architecture

### The Four Memory Stores

Every production agentic system needs to think across four distinct memory layers, each with a different scope, latency, and persistence characteristic:

```
┌────────────────────────────────────────────────────────────┐
│                   AGENT MEMORY ARCHITECTURE                │
├─────────────────┬───────────────┬────────────┬─────────────┤
│   IN-CONTEXT    │   EXTERNAL    │  EPISODIC  │  SEMANTIC   │
│  (Working Mem)  │  (Long-Term)  │  (Events)  │  (Facts)    │
├─────────────────┼───────────────┼────────────┼─────────────┤
│ Context window  │ Vector DB /   │ Event log  │ Knowledge   │
│ Token budget    │ PostgreSQL    │ Session DB │ Base / RAG  │
│ Current turn    │ Redis cache   │ Timelines  │ Embeddings  │
├─────────────────┼───────────────┼────────────┼─────────────┤
│ Fastest         │ Fast          │ Moderate   │ Moderate    │
│ Most expensive  │ Cheap         │ Cheap      │ Cheap       │
│ Volatile        │ Persistent    │ Persistent │ Persistent  │
└─────────────────┴───────────────┴────────────┴─────────────┘
```

**In-Context (Working Memory)**
The LLM's context window at any given moment. Everything the model "knows" for this inference call. The most constrained and expensive resource in the system.

**External / Long-Term Memory**
Persisted outside the model — vector stores, relational DBs, key-value caches. Accessed via retrieval (RAG, tool calls). Effectively unbounded in size.

**Episodic Memory**
A log of what happened — past agent runs, tool call sequences, user interactions, outcomes. Enables the agent to reason about its own history. Typically stored as structured event records.

**Semantic Memory**
Factual world knowledge — domain documents, ontologies, reference data. The corpus your RAG pipeline retrieves from. Static or slowly updated.

---

### Memory Access Patterns

```
Agent Turn N
     │
     ├── Retrieve from Semantic Memory (RAG)      → inject into context
     ├── Retrieve from Episodic Memory (past runs) → inject summary
     ├── Read from External KV store (user prefs)  → inject as system block
     │
     ▼
  LLM Inference  ←── Working Memory (context window)
     │
     ├── Write tool outputs → External store
     ├── Write turn summary → Episodic store
     └── Update user state  → External KV
```

---

### Choosing the Right Store

| What you need to remember | Store |
|---|---|
| Current task state, tool results this turn | In-context (working memory) |
| User preferences, persistent profile | External KV (Redis / PostgreSQL) |
| What happened in past sessions | Episodic DB (PostgreSQL event log) |
| Domain knowledge, documents | Semantic store (vector DB / pgvector) |
| Frequently accessed reference data | Cache layer (Redis, in-memory) |

---

## 2. Working Memory Management and Budgeting

Working memory is the context window — the single most constrained resource in any agentic system. Poor management leads to context overflow, silent truncation, and degraded reasoning quality well before the hard limit is hit.

### 2.1 The Token Budget Model

Think of the context window as a **fixed budget** that must be allocated across competing consumers:

```
┌─────────────────────────────────────────────────────────────────┐
│                  CONTEXT WINDOW BUDGET (e.g. 200K tokens)       │
├──────────────────┬──────────────────────────────────────────────┤
│ RESERVED (fixed) │ DYNAMIC (per-turn allocation)                │
├──────────────────┼──────────────────────────────────────────────┤
│ System prompt    │ Retrieved RAG chunks          (~10–15%)      │
│ (~5–10%)         │ Tool schemas                  (~5%)          │
│                  │ Conversation / scratchpad     (~20–30%)      │
│                  │ Tool call results             (~20–30%)      │
│                  │ Output buffer (generation)    (~10–15%)      │
└──────────────────┴──────────────────────────────────────────────┘
```

**Key principle:** Always reserve headroom. A model operating near its context limit degrades in reasoning quality (the "lost in the middle" problem) long before it hits a hard error.

---

### 2.2 Budget Allocation Strategy

```python
class TokenBudget:
    def __init__(self, model_context_limit: int):
        self.total          = model_context_limit
        self.system_prompt  = 0
        self.tool_schemas   = 0
        self.output_reserve = 2000   # always reserve for generation

    def remaining(self) -> int:
        used = self.system_prompt + self.tool_schemas + self.output_reserve
        return self.total - used

    def allocate(self, priorities: dict[str, float]) -> dict[str, int]:
        """
        priorities: {"rag_chunks": 0.35, "history": 0.30, "tool_results": 0.35}
        Proportionally allocate remaining tokens by priority weights.
        """
        budget = self.remaining()
        return {k: int(budget * v) for k, v in priorities.items()}
```

---

### 2.3 The "Lost in the Middle" Problem

LLMs attend most strongly to content at the **beginning and end** of the context window. Information buried in the middle receives less attention and is more likely to be ignored.

```
Attention weight across context position:

High  │▓▓▓▓▓                              ▓▓▓▓▓
      │     ▓▓▓▓                       ▓▓▓
      │         ▓▓▓               ▓▓▓
Low   │             ▓▓▓▓▓▓▓▓▓▓▓▓▓
      └────────────────────────────────────────→
      START                                  END
```

**Mitigation strategies:**
- Place the most critical context (task instructions, key facts) at the **start and end** of the context
- Place verbose but lower-priority content (raw tool outputs, background docs) in the middle
- Summarise middle sections aggressively before injecting
- Limit total context fill to ~70% of the window maximum

---

### 2.4 Tool Result Management

Tool calls are the biggest source of context bloat in agentic systems. A single database query or API response can consume thousands of tokens.

```python
def manage_tool_result(result: str, budget: int,
                       summarise_fn=None) -> str:
    """
    Fit a tool result into a token budget.
    Falls back to LLM summarisation if result exceeds budget.
    """
    token_count = count_tokens(result)

    if token_count <= budget:
        return result                          # fits — use as-is

    if summarise_fn and token_count <= budget * 3:
        return summarise_fn(result, budget)    # moderate overrun — summarise

    # Large overrun — truncate with marker
    truncated = truncate_to_tokens(result, budget - 50)
    return truncated + "\n\n[... result truncated — full output in external store]"
```

**Best practices:**
- Never dump raw API/DB responses directly into context
- Extract only the fields the agent actually needs
- Store full results externally; inject only a structured summary
- Use structured extraction (JSON schema) to force compact output from tools

---

### 2.5 Conversation History Pruning

Multi-turn conversations accumulate fast. Three strategies, in increasing aggressiveness:

**Strategy 1 — Sliding Window**
Keep only the last N turns. Simple, predictable, loses early context.

```python
def sliding_window(history: list[dict], max_turns: int = 10) -> list[dict]:
    return history[-max_turns * 2:]   # *2 for user+assistant pairs
```

**Strategy 2 — Token-Aware Truncation**
Evict oldest turns until the history fits the budget.

```python
def token_aware_truncate(history: list[dict], budget: int) -> list[dict]:
    kept = []
    used = 0
    for message in reversed(history):
        tokens = count_tokens(message["content"])
        if used + tokens > budget:
            break
        kept.insert(0, message)
        used += tokens
    return kept
```

**Strategy 3 — Progressive Summarisation**
Summarise old turns into a rolling summary block; keep recent turns verbatim. Covered in §3.

---

### 2.6 Context Window Hygiene Rules

- **Never let context exceed 70–75% of the model limit** — reasoning quality degrades before truncation errors occur
- **Count tokens before every LLM call** — don't estimate; use `tiktoken` or the model's tokeniser
- **Instrument and alert** — log context usage per turn; alert when approaching thresholds
- **Separate concerns in the context** — use clear delimiters between system, retrieved context, history, and current task; models reason better with structured context

---

## 3. Memory Summarization and Write Strategies

Summarisation bridges working memory and long-term memory — it compresses what happened into a form that can be cheaply re-injected later without consuming the full original token cost.

### 3.1 When to Summarise

```
Trigger                          Action
────────────────────────────────────────────────────────
Context > 70% full               Summarise oldest N turns
Task / sub-task completes        Write task summary to episodic store
Session ends                     Write session summary to user profile
Agent hands off to another agent Write handoff brief
Periodic (every K turns)         Rolling summary update
```

---

### 3.2 Progressive (Rolling) Summarisation

The most important pattern for long-running agents. Instead of keeping the full conversation, maintain a **rolling summary** that is updated incrementally:

```
Turn 1–5:   [Full verbatim history]
            ↓ (context > threshold)
Turn 6:     [Summary of turns 1–5] + [Turns 6 verbatim]
            ↓ (context > threshold again)
Turn 11:    [Summary of turns 1–10] + [Turns 11 verbatim]
            ↓
Turn 16:    [Summary of turns 1–15] + [Turns 16 verbatim]
```

```python
SUMMARISE_PROMPT = """
You are summarising a conversation segment for an AI agent's memory.

Preserve:
- All decisions made
- All tool calls and their outcomes
- All user-stated preferences or constraints
- Any unresolved questions or pending actions

Discard:
- Pleasantries, filler, and meta-commentary
- Redundant restatements
- Intermediate reasoning that led to a discarded path

Previous summary (if any):
{previous_summary}

New turns to incorporate:
{new_turns}

Write an updated, consolidated summary in past tense.
"""

def rolling_summarise(previous_summary: str,
                      new_turns: list[dict],
                      llm) -> str:
    prompt = SUMMARISE_PROMPT.format(
        previous_summary=previous_summary or "None",
        new_turns=format_turns(new_turns),
    )
    return llm.invoke(prompt)
```

---

### 3.3 Hierarchical Summarisation

For very long agent runs (hundreds of turns), a single rolling summary becomes stale and lossy. Use a **two-level hierarchy**:

```
Level 1 — Turn summaries      (per 5–10 turns, ~100 tokens each)
Level 2 — Session summary     (per session, ~300–500 tokens)
Level 3 — User/task profile   (persistent, ~200 tokens, updated on change)

At retrieval time:
  → Always inject: Level 3 (user profile)
  → Conditionally inject: Level 2 (recent session summary)
  → Rarely inject: Level 1 (only if directly relevant to current task)
```

---

### 3.4 Write Strategies for External Memory

Not everything should be written back to long-term memory. Apply a **write filter**:

```python
WRITE_DECISION_PROMPT = """
Evaluate this agent turn. Decide what (if anything) should be 
written to long-term memory.

Categories:
  USER_PREFERENCE  — stated preference about behaviour, format, topics
  DECISION         — a confirmed decision made by user or agent
  TASK_OUTCOME     — result of a completed task or sub-task
  CONSTRAINT       — a rule or constraint to remember
  DISCARD          — nothing worth persisting

Turn content:
{turn}

Respond as JSON:
{{"category": "...", "persist": true/false, "summary": "..."}}
"""

def evaluate_for_write(turn: str, llm) -> dict:
    response = llm.invoke(WRITE_DECISION_PROMPT.format(turn=turn))
    return json.loads(response)
```

**Write routing by category:**

| Category | Write destination |
|---|---|
| `USER_PREFERENCE` | User profile store (PostgreSQL / Redis) |
| `DECISION` | Episodic event log |
| `TASK_OUTCOME` | Task result store; update task state |
| `CONSTRAINT` | System prompt augmentation store |
| `DISCARD` | Nothing written |

---

### 3.5 Memory Decay and Freshness

Not all memories should live forever. Apply a **TTL (time-to-live) and relevance decay** model:

```
Memory Type          TTL / Retention Policy
──────────────────────────────────────────────────────────
Turn-level summary   24–48 hours (session scope)
Session summary      30 days (unless reinforced by re-access)
User preference      Indefinite (until explicitly changed)
Task outcome         Duration of project / task lifecycle
Semantic (RAG docs)  Versioned; expire on document update
```

```python
def write_with_ttl(store, key: str, value: dict, ttl_days: int):
    value["expires_at"] = (datetime.utcnow()
                           + timedelta(days=ttl_days)).isoformat()
    store.set(key, json.dumps(value))
    # In Redis: store.expire(key, ttl_days * 86400)
```

---

### 3.6 The REMEMBER / RECALL / FORGET Pattern

A clean interface pattern for agent memory operations, usable as tool definitions in an agentic framework:

```python
# Three memory tools exposed to the agent

REMEMBER = {
    "name": "remember",
    "description": "Persist a fact, decision, or preference to long-term memory.",
    "parameters": {
        "category": "USER_PREFERENCE | DECISION | TASK_OUTCOME | CONSTRAINT",
        "content":  "The information to persist, concisely stated.",
        "ttl_days": "How long to retain (0 = indefinite).",
    }
}

RECALL = {
    "name": "recall",
    "description": "Retrieve relevant memories given a query.",
    "parameters": {
        "query":   "Natural language description of what to retrieve.",
        "limit":   "Max number of memories to return (default 5).",
    }
}

FORGET = {
    "name": "forget",
    "description": "Delete or invalidate a stored memory.",
    "parameters": {
        "memory_id": "ID of the memory to remove.",
        "reason":    "Why this memory is no longer valid.",
    }
}
```

---

### 3.7 Summary of Key Principles

| Principle | Guidance |
|---|---|
| **Budget first** | Always know your token budget before composing a context |
| **Retrieve, don't store** | Keep working memory lean; pull from external stores on demand |
| **Summarise at boundaries** | Sub-task end, session end, context threshold — all are write triggers |
| **Write with intent** | Not every turn deserves persistence; classify before writing |
| **Position matters** | Critical information at start/end of context; verbose content in middle |
| **Instrument everything** | Token usage, memory hit rates, and summarisation quality are observable metrics |
| **Decay is healthy** | Stale memories degrade agent performance; apply TTLs aggressively |
