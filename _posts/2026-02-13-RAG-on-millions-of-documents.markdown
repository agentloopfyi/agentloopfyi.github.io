---
layout: post
comments: false
title:  "RAG on Millions of documents"
date:   2026-02-13 10:00:00
---

# How to build Enterprise Hybrid RAG
*Article crafted from experience, then written down using AI*

### PostgreSQL · pgvector · BM25/FTS · RRF · Cohere Rerank · GPT-5

#### For 1M+ Document Scale · Python 3.12+ · LangChain LCEL & Pure-Python Approaches

---

## Table of Contents

1. [What is Hybrid RAG and Why?](#1-what-is-hybrid-rag-and-why)
2. [Technology Stack](#2-technology-stack)
3. [Core Concepts](#3-core-concepts)
   - 3.1 Dense Retrieval (pgvector / ANN)
   - 3.2 Lexical Retrieval (BM25 / PostgreSQL FTS)
   - 3.3 Reciprocal Rank Fusion (RRF)
   - 3.4 Cohere Reranking
4. [Full System Architecture](#4-full-system-architecture)
5. [PostgreSQL Schema & Indexes](#5-postgresql-schema--indexes)
6. [Ingestion Pipeline](#6-ingestion-pipeline)
7. [Retrieval Pipeline — Pure Python (Non-LangChain)](#7-retrieval-pipeline--pure-python-non-langchain)
8. [Retrieval Pipeline — LangChain LCEL](#8-retrieval-pipeline--langchain-lcel)
9. [Prompt Design & Grounded Generation](#9-prompt-design--grounded-generation)
10. [Production Configuration & Sizing](#10-production-configuration--sizing)
11. [Corrections & Notes on the Original Material](#11-corrections--notes-on-the-original-material)
12. [Quick Reference Cheat Sheet](#12-quick-reference-cheat-sheet)

---

## 1. What is Hybrid RAG and Why?

**Retrieval-Augmented Generation (RAG)** grounds an LLM's answer in retrieved documents, preventing hallucination and enabling access to private or current knowledge.

**Why "hybrid"?** No single retrieval method is optimal for all queries:

| Retrieval Type | Strengths | Weaknesses |
|---|---|---|
| **Dense (vector)** | Semantic similarity, paraphrase matching, multilingual | Misses exact keyword matches; computationally expensive to build |
| **Lexical (BM25/FTS)** | Exact term matches, product codes, names, IDs | No semantic understanding; vocabulary mismatch problem |
| **Hybrid (both)** | Best of both worlds | Requires merging and reranking stages |

**The production-grade pipeline** adds two further stages after retrieval:

- **RRF (Reciprocal Rank Fusion)** — merges ranked lists from both retrievers into a single ranked list without needing score calibration.
- **Cross-encoder Reranking (Cohere)** — re-scores the top-N candidates using a (query, document) pair model for much higher precision before sending to the LLM.

---

## 2. Technology Stack

| Layer | Technology | Notes |
|---|---|---|
| LLM | GPT-5 (`gpt-5`) | Temperature = 0 for grounded QA |
| Embedding | `text-embedding-3-large` | Dimension = 3072 |
| Vector DB | PostgreSQL + pgvector | HNSW index for ANN at 10M+ scale |
| Lexical Search | PostgreSQL Full Text Search | BM25-approximate via `ts_rank`; see §3.2 |
| Hybrid Merge | Reciprocal Rank Fusion (RRF) | k=60 is the standard constant |
| Reranking | Cohere `rerank-v3.5` | Cross-encoder; far higher precision than bi-encoder |
| Orchestration | Python 3.12+ / LangChain LCEL | Two variants covered |
| Storage | PostgreSQL (chunks + metadata) | Single store for simplicity |

---

## 3. Core Concepts

### 3.1 Dense Retrieval (pgvector / ANN)

Chunks are embedded into high-dimensional vectors. At query time, the query is also embedded, and we find the nearest neighbours by **cosine similarity**.

```
Query: "What causes inflation?"
           │
     Embedding Model
           │
     [0.12, -0.87, 0.44 ... ]   (3072-dim vector)
           │
   pgvector HNSW index
           │
   Approximate Nearest Neighbours
           │
   Top-100 semantically similar chunks
```

**pgvector operators:**

| Operator | Metric |
|---|---|
| `<=>` | Cosine distance (use for normalised embeddings) |
| `<->` | L2 (Euclidean) distance |
| `<#>` | Negative inner product |

For `text-embedding-3-large`, use **cosine distance (`<=>`)**.

**HNSW vs IVFFlat:**

| Index | 10M Scale | Build Time | Query Speed |
|---|---|---|---|
| **HNSW** | ✅ Recommended | Slower | Very fast, high recall |
| IVFFlat | Needs careful tuning | Faster | Acceptable |

---

### 3.2 Lexical Retrieval — PostgreSQL FTS vs True BM25

> ⚠️ **Important clarification:** The original material references "pg_textsearch (BM25)". This needs disambiguation:

**PostgreSQL's built-in FTS** (`tsvector` / `tsquery` / `ts_rank`) is **NOT true BM25**. It uses a scoring function that approximates relevance ranking but does not implement the full BM25 formula (which accounts for term frequency saturation and document length normalisation).

**Options for true BM25 in PostgreSQL:**

| Extension | True BM25? | Notes |
|---|---|---|
| **pg_bm25 / ParadeDB** | ✅ Yes | `CREATE INDEX USING bm25`; Tantivy-based |
| **PostgreSQL built-in FTS** | ❌ No (approximation) | `ts_rank` — fine for most use cases |
| **Elasticsearch** | ✅ Yes | External service; operationally heavier |

**For this guide**, we cover both:

- **Built-in FTS** (`ts_rank`) — simpler, no extra extension
- **pg_bm25 / ParadeDB** — true BM25, drop-in SQL syntax

**How PostgreSQL FTS works:**

```sql
-- Text → tsvector (lexemes)
SELECT to_tsvector('english', 'The quick brown fox jumps');
-- Result: 'brown':3 'fox':4 'jump':5 'quick':2

-- Query parsing
SELECT plainto_tsquery('english', 'quick fox');
-- Result: 'quick' & 'fox'

-- Match + rank
SELECT chunk_text, ts_rank(search_vector, plainto_tsquery('english', 'quick fox')) AS score
FROM document_chunks
WHERE search_vector @@ plainto_tsquery('english', 'quick fox')
ORDER BY score DESC;
```

**True BM25 with pg_bm25 (ParadeDB):**

```sql
-- Install extension (ParadeDB distribution)
CREATE EXTENSION IF NOT EXISTS pg_bm25;

-- Create BM25 index
CREATE INDEX chunks_bm25_idx ON document_chunks
USING bm25(id, chunk_text)
WITH (key_field='id', text_fields='{"chunk_text": {}}');

-- BM25 search
SELECT id, chunk_text, paradedb.score(id)
FROM document_chunks
WHERE chunk_text @@@ 'quick fox'
ORDER BY paradedb.score(id) DESC
LIMIT 100;
```

---

### 3.3 Reciprocal Rank Fusion (RRF)

RRF is a **rank aggregation algorithm** that combines multiple ranked lists without needing calibrated scores.

**Formula:**

```
RRF_score(doc d) = Σ  1 / (k + rank_i(d))
                  i

Where:
  k    = smoothing constant (default 60)
  rank_i(d) = rank of document d in list i (1-indexed)
```

**Why k=60?** It was empirically determined in the original 2009 paper (Cormack et al.) to work well across diverse retrieval tasks. It dampens the extreme influence of the very top rank.

**Example walkthrough:**

```
Dense Results (top-5):     Lexical Results (top-5):
  Rank 1 → Doc A             Rank 1 → Doc C
  Rank 2 → Doc B             Rank 2 → Doc A
  Rank 3 → Doc C             Rank 3 → Doc D
  Rank 4 → Doc D             Rank 4 → Doc E
  Rank 5 → Doc E             Rank 5 → Doc B

RRF scores (k=60):
  Doc A: 1/(60+1) + 1/(60+2) = 0.01639 + 0.01613 = 0.03252  ← Highest
  Doc C: 1/(60+3) + 1/(60+1) = 0.01587 + 0.01639 = 0.03226
  Doc B: 1/(60+2) + 1/(60+5) = 0.01613 + 0.01538 = 0.03151
  Doc D: 1/(60+4) + 1/(60+3) = 0.01563 + 0.01587 = 0.03150
  Doc E: 1/(60+5) + 1/(60+4) = 0.01538 + 0.01563 = 0.03101

Final merged order: A → C → B → D → E
```

---

### 3.4 Cohere Reranking

After RRF, we have ~100–150 candidates. Most are relevant but not precisely ranked. A **cross-encoder reranker** fixes this.

**Bi-encoder (embedding retrieval) vs Cross-encoder (reranker):**

```
Bi-encoder (fast, approximate):
  Query ──→ Encoder ──→ q_vec
  Doc   ──→ Encoder ──→ d_vec
                          ↓
                    cosine_sim(q_vec, d_vec)
  [Encodes independently — fast but less accurate]

Cross-encoder (slow, precise):
  [Query + Doc] ──→ Encoder ──→ Relevance Score
  [Sees both together — deeply understands interaction]
```

The cross-encoder considers the full query-document interaction, catching nuances that bi-encoders miss. It's too slow for full-corpus search, but perfect for re-scoring 100–150 candidates.

```
Before Rerank (RRF top-10 example):
  1. "France has 68 million people"
  2. "Paris is the capital of France"        ← Should be #1
  3. "Germany borders France to the east"
  ...

After Cohere Rerank (query: "What is the capital of France?"):
  1. "Paris is the capital of France"        ✅
  2. "France has 68 million people"
  3. "Germany borders France to the east"
```

---

## 4. Full System Architecture

### 4.1 End-to-End Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         INGESTION PATH                          │
│                                                                 │
│  Raw Documents                                                  │
│       │                                                         │
│       ▼                                                         │
│  Chunker (300–500 tokens, 20% overlap)                          │
│       │                                                         │
│       ├──────────────────────────┐                              │
│       │                          │                              │
│       ▼                          ▼                              │
│  OpenAI Embeddings          to_tsvector()                       │
│  text-embedding-3-large     (FTS lexemes)                       │
│  (dim=3072)                                                     │
│       │                          │                              │
│       └──────────────┬───────────┘                              │
│                      ▼                                          │
│             PostgreSQL (document_chunks)                        │
│             ├── embedding  (vector/HNSW)                        │
│             ├── search_vector (tsvector/GIN)                    │
│             └── metadata (JSONB)                                │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                          QUERY PATH                             │
│                                                                 │
│  User Question                                                  │
│       │                                                         │
│       ▼                                                         │
│  [Optional] Query Rewrite (GPT-5)                               │
│  — Disambiguate, expand acronyms, fix typos                     │
│       │                                                         │
│       ▼                                                         │
│  [Optional] Multi-Query Expansion                               │
│  — Generate 3–5 variations for recall boost                     │
│       │                                                         │
│       ├──────────────────────────┐                              │
│       │                          │                              │
│       ▼                          ▼                              │
│  Dense Search               Lexical Search                      │
│  (pgvector HNSW)            (PostgreSQL FTS)                    │
│  Top-100 chunks             Top-100 chunks                      │
│       │                          │                              │
│       └──────────────┬───────────┘                              │
│                      ▼                                          │
│           Reciprocal Rank Fusion (RRF)                          │
│                      │                                          │
│               Top 100–150 chunks                                │
│                      │                                          │
│                      ▼                                          │
│            Cohere Rerank API (rerank-v3.5)                      │
│                      │                                          │
│                 Top 10 chunks                                   │
│                      │                                          │
│                      ▼                                          │
│            Context Assembly + Citations                         │
│                      │                                          │
│                      ▼                                          │
│             GPT-5 (temperature=0)                               │
│                      │                                          │
│                      ▼                                          │
│        Grounded Answer with Chunk Citations                     │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Retrieval Stage Sizing

| Stage | Count | Rationale |
|---|---|---|
| Dense Search candidates | 100 | High recall; ANN is fast enough |
| Lexical Search candidates | 100 | High recall for keyword matching |
| After RRF merge | 100–150 | Union of both result sets |
| After Cohere Rerank | 10–20 | High precision shortlist |
| Sent to GPT-5 prompt | 5–10 | Fits context window; avoids noise |

---

## 5. PostgreSQL Schema & Indexes

### 5.1 Extension Setup

```sql
-- pgvector extension (must be installed on server first)
CREATE EXTENSION IF NOT EXISTS vector;

-- Optional: true BM25 via ParadeDB
-- CREATE EXTENSION IF NOT EXISTS pg_bm25;
```

### 5.2 Document Hierarchy (Recommended)

For large corpora, maintain a three-level hierarchy:

```sql
-- Level 1: Source documents
CREATE TABLE documents (
    id          BIGSERIAL PRIMARY KEY,
    source_uri  TEXT NOT NULL,
    title       TEXT,
    doc_type    TEXT,                    -- 'pdf', 'html', 'docx', etc.
    ingested_at TIMESTAMPTZ DEFAULT NOW(),
    metadata    JSONB
);

-- Level 2: Sections / headings (optional but useful for navigation)
CREATE TABLE document_sections (
    id          BIGSERIAL PRIMARY KEY,
    document_id BIGINT REFERENCES documents(id) ON DELETE CASCADE,
    section_num INT,
    heading     TEXT,
    metadata    JSONB
);

-- Level 3: Chunks (the retrieval unit)
CREATE TABLE document_chunks (
    id            BIGSERIAL PRIMARY KEY,
    document_id   BIGINT REFERENCES documents(id) ON DELETE CASCADE,
    section_id    BIGINT REFERENCES document_sections(id),
    chunk_number  INT NOT NULL,
    chunk_text    TEXT NOT NULL,
    token_count   INT,
    embedding     vector(3072),          -- text-embedding-3-large output
    search_vector TSVECTOR,              -- FTS index column
    metadata      JSONB,                 -- arbitrary key-value pairs
    created_at    TIMESTAMPTZ DEFAULT NOW()
);
```

### 5.3 Indexes

```sql
-- ── Vector Index (HNSW) ──────────────────────────────────────────
-- HNSW is preferred over IVFFlat at 10M+ scale.
-- m=16, ef_construction=64 are good starting defaults.
-- Tune ef_search at query time for recall/latency tradeoff.

CREATE INDEX idx_chunks_embedding
ON document_chunks
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- ── Full Text Search Index (GIN) ──────────────────────────────────
CREATE INDEX idx_chunks_fts
ON document_chunks
USING gin(search_vector);

-- ── Supporting Indexes ────────────────────────────────────────────
CREATE INDEX idx_chunks_document_id ON document_chunks(document_id);
CREATE INDEX idx_chunks_metadata    ON document_chunks USING gin(metadata);

-- ── BM25 Index (ParadeDB / pg_bm25 only) ─────────────────────────
-- Uncomment if using ParadeDB:
-- CREATE INDEX idx_chunks_bm25
-- ON document_chunks
-- USING bm25(id, chunk_text)
-- WITH (key_field='id', text_fields='{"chunk_text": {}}');
```

### 5.4 Lexical Search Indexing — FTS Trigger vs BM25

The two lexical retrieval paths have **fundamentally different indexing mechanisms**. Choose one path and apply only its schema/index/query pattern. Do not mix them.

| | PostgreSQL FTS (`ts_rank`) | pg_bm25 / ParadeDB (true BM25) |
|---|---|---|
| **Extra column needed?** | ✅ Yes — `search_vector TSVECTOR` | ❌ No — index is on `chunk_text` directly |
| **Trigger needed?** | ✅ Yes — to keep `search_vector` in sync | ❌ No — index maintains itself |
| **Index type** | GIN on `search_vector` | BM25 on `chunk_text` (Tantivy engine) |
| **Query operator** | `@@` with `plainto_tsquery` | `@@@` with plain string |
| **Scoring function** | `ts_rank` (approximation) | `paradedb.score(id)` (true BM25) |
| **When to use** | Simpler setup, no extra extension | When precise BM25 scoring is required |

---

#### Path A — PostgreSQL FTS (trigger required)

The `search_vector` column is a pre-computed `TSVECTOR`. A trigger keeps it automatically in sync whenever `chunk_text` is inserted or updated.

```sql
-- ── Trigger function ──────────────────────────────────────────────
CREATE OR REPLACE FUNCTION update_search_vector()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector := to_tsvector('english', COALESCE(NEW.chunk_text, ''));
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- ── Attach trigger to the table ───────────────────────────────────
-- Fires BEFORE INSERT or any UPDATE that touches chunk_text.
-- No manual population needed — every write is handled automatically.
CREATE TRIGGER trg_update_search_vector
BEFORE INSERT OR UPDATE OF chunk_text
ON document_chunks
FOR EACH ROW EXECUTE FUNCTION update_search_vector();
```

With this trigger in place, your INSERT statement needs no `to_tsvector()` call — the column is filled automatically:

```sql
-- ✅ Clean insert — trigger handles search_vector
INSERT INTO document_chunks (document_id, chunk_number, chunk_text, embedding, metadata)
VALUES (%s, %s, %s, %s, %s);

-- ❌ Unnecessary — don't pass search_vector manually when the trigger is active
-- INSERT INTO document_chunks (..., search_vector) VALUES (..., to_tsvector('english', %s));
```

Query using FTS:

```sql
SELECT
    id,
    chunk_text,
    ts_rank(search_vector, plainto_tsquery('english', %s), 32) AS score
FROM document_chunks
WHERE search_vector @@ plainto_tsquery('english', %s)
ORDER BY score DESC
LIMIT 100;
```

---

#### Path B — pg_bm25 / ParadeDB (no trigger, no extra column)

pg_bm25 builds its BM25 index **directly on the raw `chunk_text` column** using the Tantivy search engine underneath. There is no `search_vector` column, no `TSVECTOR`, and no trigger involved at all.

```sql
-- ── Extension (requires ParadeDB PostgreSQL distribution) ─────────
CREATE EXTENSION IF NOT EXISTS pg_bm25;

-- ── BM25 index on the raw text column ────────────────────────────
-- key_field:   the primary key column (for score retrieval)
-- text_fields: which columns to index for full-text BM25 search
CREATE INDEX idx_chunks_bm25
ON document_chunks
USING bm25(id, chunk_text)
WITH (key_field='id', text_fields='{"chunk_text": {}}');
```

With pg_bm25, inserts are identical to any normal insert — no special handling:

```sql
-- ✅ Normal insert — pg_bm25 index updates automatically (like any B-tree)
INSERT INTO document_chunks (document_id, chunk_number, chunk_text, embedding, metadata)
VALUES (%s, %s, %s, %s, %s);
```

Query using true BM25:

```sql
-- @@@ is the ParadeDB full-text match operator
-- paradedb.score(id) returns the BM25 relevance score for each row
SELECT id, chunk_text, paradedb.score(id) AS bm25_score
FROM document_chunks
WHERE chunk_text @@@ %s          -- plain query string, no tsquery conversion
ORDER BY paradedb.score(id) DESC
LIMIT 100;
```

---

#### Schema impact of each path

If using **Path A (FTS)**, your `document_chunks` table includes `search_vector`:

```sql
CREATE TABLE document_chunks (
    id            BIGSERIAL PRIMARY KEY,
    document_id   BIGINT REFERENCES documents(id) ON DELETE CASCADE,
    section_id    BIGINT REFERENCES document_sections(id),
    chunk_number  INT NOT NULL,
    chunk_text    TEXT NOT NULL,
    token_count   INT,
    embedding     vector(3072),
    search_vector TSVECTOR,        -- ← FTS path only
    metadata      JSONB,
    created_at    TIMESTAMPTZ DEFAULT NOW()
);
```

If using **Path B (pg_bm25)**, drop `search_vector` entirely — it serves no purpose:

```sql
CREATE TABLE document_chunks (
    id            BIGSERIAL PRIMARY KEY,
    document_id   BIGINT REFERENCES documents(id) ON DELETE CASCADE,
    section_id    BIGINT REFERENCES document_sections(id),
    chunk_number  INT NOT NULL,
    chunk_text    TEXT NOT NULL,   -- ← BM25 index built directly on this
    token_count   INT,
    embedding     vector(3072),
    metadata      JSONB,
    created_at    TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 6. Ingestion Pipeline

### 6.1 Python Dependencies

```bash
pip install psycopg2-binary          # PostgreSQL driver
pip install openai                   # OpenAI SDK (embeddings + GPT-5)
pip install cohere                   # Cohere reranking
pip install tiktoken                 # Token counting
pip install langchain                # Optional: LCEL orchestration
pip install langchain-openai         # Optional: LangChain OpenAI integration
pip install numpy                    # RRF computation
```

### 6.2 Configuration

```python
import os
import psycopg2
import openai
import cohere
import tiktoken

# ── Clients ───────────────────────────────────────────────────────
openai_client = openai.OpenAI(api_key=os.environ["OPENAI_API_KEY"])
cohere_client = cohere.Client(api_key=os.environ["COHERE_API_KEY"])

DB_CONFIG = {
    "host": os.environ.get("PG_HOST", "localhost"),
    "port": int(os.environ.get("PG_PORT", 5432)),
    "dbname": os.environ.get("PG_DB", "ragdb"),
    "user": os.environ.get("PG_USER", "postgres"),
    "password": os.environ.get("PG_PASSWORD", ""),
}

EMBEDDING_MODEL   = "text-embedding-3-large"
EMBEDDING_DIM     = 3072
RERANK_MODEL      = "rerank-v3.5"
LLM_MODEL         = "gpt-5"
CHUNK_SIZE_TOKENS = 400   # target tokens per chunk
CHUNK_OVERLAP     = 0.20  # 20% overlap
```

### 6.3 Chunker

```python
def chunk_text(text: str, max_tokens: int = CHUNK_SIZE_TOKENS,
               overlap_ratio: float = CHUNK_OVERLAP) -> list[str]:
    """
    Token-aware sliding window chunker.
    Returns a list of text chunks.
    """
    enc = tiktoken.encoding_for_model("gpt-4o")  # cl100k_base
    tokens = enc.encode(text)
    overlap = int(max_tokens * overlap_ratio)
    step    = max_tokens - overlap

    chunks = []
    start  = 0
    while start < len(tokens):
        end        = min(start + max_tokens, len(tokens))
        chunk_toks = tokens[start:end]
        chunks.append(enc.decode(chunk_toks))
        if end == len(tokens):
            break
        start += step

    return chunks
```

### 6.4 Embedding Generation

```python
def embed(texts: list[str]) -> list[list[float]]:
    """
    Batch embed a list of texts using text-embedding-3-large.
    OpenAI supports up to 2048 inputs per call; keep batches ≤ 500
    for safety.
    """
    response = openai_client.embeddings.create(
        model=EMBEDDING_MODEL,
        input=texts,
    )
    return [item.embedding for item in response.data]


def embed_single(text: str) -> list[float]:
    return embed([text])[0]
```

### 6.5 Inserting Chunks

```python
INSERT_SQL = """
INSERT INTO document_chunks
    (document_id, chunk_number, chunk_text, token_count, embedding, metadata)
VALUES
    (%s, %s, %s, %s, %s, %s)
ON CONFLICT DO NOTHING
"""

def ingest_document(conn, document_id: int, text: str,
                    metadata: dict = None):
    """
    Chunk, embed, and insert a document's chunks into PostgreSQL.
    The search_vector column is handled by the trigger.
    """
    enc    = tiktoken.encoding_for_model("gpt-4o")
    chunks = chunk_text(text)

    # Embed all chunks in one batch call for efficiency
    embeddings = embed(chunks)

    with conn.cursor() as cur:
        for i, (chunk, embedding) in enumerate(zip(chunks, embeddings)):
            token_count = len(enc.encode(chunk))
            import json
            cur.execute(INSERT_SQL, (
                document_id,
                i,
                chunk,
                token_count,
                embedding,            # psycopg2 serialises Python list → pgvector
                json.dumps(metadata or {}),
            ))
    conn.commit()
    print(f"Ingested {len(chunks)} chunks for document {document_id}")
```

---

## 7. Retrieval Pipeline — Pure Python (Non-LangChain)

### 7.1 Dense Search (pgvector)

```python
def dense_search(conn, query_embedding: list[float],
                 k: int = 100) -> list[tuple]:
    """
    Approximate nearest neighbour search using HNSW cosine distance.
    Returns list of (id, chunk_text) tuples.

    Note: set ef_search for query-time recall tuning:
      SET hnsw.ef_search = 200;   -- higher = better recall, slower
    """
    with conn.cursor() as cur:
        # Optionally set ef_search for this session
        cur.execute("SET hnsw.ef_search = 200;")

        cur.execute("""
            SELECT
                id,
                chunk_text,
                1 - (embedding <=> %s::vector) AS cosine_similarity
            FROM document_chunks
            ORDER BY embedding <=> %s::vector
            LIMIT %s
        """, (query_embedding, query_embedding, k))

        return cur.fetchall()   # [(id, chunk_text, similarity), ...]
```

### 7.2 Lexical Search (PostgreSQL FTS)

```python
def lexical_search(conn, query: str, k: int = 100) -> list[tuple]:
    """
    Full-text search using PostgreSQL tsvector + ts_rank.
    ts_rank is a BM25 approximation — good enough for most workloads.

    For true BM25, replace this with a pg_bm25 / ParadeDB query.
    """
    with conn.cursor() as cur:
        cur.execute("""
            SELECT
                id,
                chunk_text,
                ts_rank(
                    search_vector,
                    plainto_tsquery('english', %s),
                    32          -- normalization: divide by doc length
                ) AS bm25_score
            FROM document_chunks
            WHERE search_vector @@ plainto_tsquery('english', %s)
            ORDER BY bm25_score DESC
            LIMIT %s
        """, (query, query, k))

        return cur.fetchall()   # [(id, chunk_text, bm25_score), ...]
```

> **True BM25 variant (pg_bm25 / ParadeDB):**
>
> ```python
> def lexical_search_bm25(conn, query: str, k: int = 100) -> list[tuple]:
>     with conn.cursor() as cur:
>         cur.execute("""
>             SELECT id, chunk_text, paradedb.score(id) AS bm25_score
>             FROM document_chunks
>             WHERE chunk_text @@@ %s
>             ORDER BY paradedb.score(id) DESC
>             LIMIT %s
>         """, (query, k))
>         return cur.fetchall()
> ```

### 7.3 Reciprocal Rank Fusion

```python
from collections import defaultdict

def rrf_merge(dense_results: list[tuple],
              lexical_results: list[tuple],
              k: int = 60) -> list[tuple]:
    """
    Reciprocal Rank Fusion.

    RRF_score(d) = Σ_i  1 / (k + rank_i(d))

    k=60 is the empirically validated default from Cormack et al., 2009.

    Parameters:
        dense_results   — list of (id, chunk_text, score) from pgvector
        lexical_results — list of (id, chunk_text, score) from FTS/BM25
        k               — smoothing constant (default 60)

    Returns:
        Merged + sorted list of (id, chunk_text, rrf_score) tuples.
    """
    scores: dict[int, float]       = defaultdict(float)
    docs:   dict[int, tuple]       = {}

    # Accumulate RRF scores from dense results (1-indexed ranks)
    for rank, row in enumerate(dense_results, start=1):
        doc_id = row[0]
        docs[doc_id] = row
        scores[doc_id] += 1.0 / (k + rank)

    # Accumulate RRF scores from lexical results
    for rank, row in enumerate(lexical_results, start=1):
        doc_id = row[0]
        docs[doc_id] = row          # lexical row overwrites; same doc_id
        scores[doc_id] += 1.0 / (k + rank)

    # Sort by descending RRF score
    merged_ids = sorted(scores, key=lambda d: scores[d], reverse=True)

    return [(doc_id, docs[doc_id][1], scores[doc_id])
            for doc_id in merged_ids]
```

### 7.4 Cohere Reranking

```python
def cohere_rerank(question: str, candidates: list[tuple],
                  top_n: int = 10) -> list[tuple]:
    """
    Rerank candidates using Cohere cross-encoder.

    candidates: list of (id, chunk_text, score) from RRF
    Returns: top_n reranked (id, chunk_text, relevance_score) tuples.
    """
    if not candidates:
        return []

    texts = [row[1] for row in candidates]

    response = cohere_client.rerank(
        query=question,
        documents=texts,
        top_n=top_n,
        model=RERANK_MODEL,
        return_documents=True,
    )

    reranked = []
    for result in response.results:
        original_row   = candidates[result.index]
        relevance_score = result.relevance_score
        reranked.append((original_row[0], original_row[1], relevance_score))

    return reranked
```

### 7.5 Context Builder

```python
def build_context(chunks: list[tuple]) -> str:
    """
    Assemble numbered context blocks for the LLM prompt.
    chunks: list of (id, chunk_text, score)
    """
    parts = []
    for chunk_id, chunk_text, _ in chunks:
        parts.append(f"[Chunk {chunk_id}]\n{chunk_text.strip()}")
    return "\n\n---\n\n".join(parts)
```

### 7.6 GPT-5 Generation

```python
SYSTEM_PROMPT = """You are a grounded question-answering assistant.

Rules:
1. Answer ONLY using the supplied context below.
2. Do NOT use any external knowledge or prior training data.
3. Cite the chunk IDs that support each statement (e.g., [Chunk 145]).
4. If the answer is not present in the context, respond exactly:
   "I don't have enough information to answer this question."
5. Be concise and factually precise.
"""

def build_prompt(question: str, context: str) -> str:
    return f"""{SYSTEM_PROMPT}

=== CONTEXT ===
{context}

=== QUESTION ===
{question}

=== ANSWER ==="""


def generate_answer(question: str, context: str) -> str:
    response = openai_client.chat.completions.create(
        model=LLM_MODEL,
        temperature=0,
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user",   "content": f"Context:\n{context}\n\nQuestion:\n{question}"},
        ],
    )
    return response.choices[0].message.content
```

### 7.7 Complete Pure-Python Pipeline

```python
def hybrid_rag_query(question: str, conn,
                     dense_k:   int = 100,
                     lexical_k: int = 100,
                     rrf_k:     int = 60,
                     rerank_n:  int = 10) -> dict:
    """
    Full hybrid RAG query pipeline (non-LangChain).

    Returns:
        {
            "question":      str,
            "answer":        str,
            "chunks_used":   list of (id, text, score),
            "context":       str,
        }
    """
    # 1. Embed the question
    query_embedding = embed_single(question)

    # 2. Dual retrieval
    dense_results   = dense_search(conn, query_embedding, k=dense_k)
    lexical_results = lexical_search(conn, question, k=lexical_k)

    # 3. Merge with RRF
    merged = rrf_merge(dense_results, lexical_results, k=rrf_k)

    # 4. Rerank with Cohere
    top_chunks = cohere_rerank(question, merged, top_n=rerank_n)

    # 5. Build context
    context = build_context(top_chunks)

    # 6. Generate answer
    answer  = generate_answer(question, context)

    return {
        "question":    question,
        "answer":      answer,
        "chunks_used": top_chunks,
        "context":     context,
    }


# ── Usage ──────────────────────────────────────────────────────────
if __name__ == "__main__":
    conn = psycopg2.connect(**DB_CONFIG)
    result = hybrid_rag_query(
        question="What is the capital of France?",
        conn=conn,
    )
    print(result["answer"])
    conn.close()
```

---

## 8. Retrieval Pipeline — LangChain LCEL

### 8.1 LCEL Primer

LangChain Expression Language (LCEL) uses the `|` (pipe) operator to chain `Runnable` objects:

```
chain = step_1 | step_2 | step_3

result = chain.invoke(input)

# Equivalent to:
# result = step_3.invoke(step_2.invoke(step_1.invoke(input)))
```

Key `Runnable` types:

| Type | Purpose |
|---|---|
| `RunnableLambda` | Wrap any Python function as a Runnable |
| `RunnableParallel` | Run multiple Runnables in parallel |
| `RunnablePassthrough` | Pass input unchanged (useful for merging) |
| `ChatPromptTemplate` | Build prompt templates |
| `ChatOpenAI` | LangChain LLM wrapper |

### 8.2 Setup

```python
from langchain_core.runnables import (
    RunnableLambda,
    RunnableParallel,
    RunnablePassthrough,
)
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_openai import ChatOpenAI, OpenAIEmbeddings

# LangChain-compatible LLM and embeddings
llm = ChatOpenAI(model=LLM_MODEL, temperature=0)
embeddings = OpenAIEmbeddings(model=EMBEDDING_MODEL)
```

### 8.3 Individual Runnable Steps

```python
# ── Step 1: Query Rewrite ──────────────────────────────────────────
rewrite_prompt = ChatPromptTemplate.from_template("""
You are a search query optimizer.
Rewrite the following question to be more precise and search-friendly.
Return ONLY the rewritten query, nothing else.

Original: {question}
Rewritten:""")

query_rewriter = (
    rewrite_prompt
    | llm
    | StrOutputParser()
)


# ── Step 2: Multi-Query Expansion ─────────────────────────────────
multi_query_prompt = ChatPromptTemplate.from_template("""
Generate 3 different search query variations for the question below.
Return one query per line. No numbering or bullets.

Question: {question}
Queries:""")

multi_query_expander = (
    multi_query_prompt
    | llm
    | StrOutputParser()
    | RunnableLambda(lambda text: [q.strip() for q in text.strip().split("\n") if q.strip()])
)


# ── Step 3: Hybrid Retriever ───────────────────────────────────────
def hybrid_retrieve(question: str, conn, dense_k=100, lexical_k=100) -> list[tuple]:
    """Runs dense + lexical search for a single query."""
    q_emb    = embed_single(question)
    dense    = dense_search(conn, q_emb, k=dense_k)
    lexical  = lexical_search(conn, question, k=lexical_k)
    return dense, lexical


def multi_query_retrieve(questions: list[str], conn) -> list[tuple]:
    """
    Runs hybrid retrieval for multiple query variants,
    then performs a single RRF over all results.
    """
    all_dense   = []
    all_lexical = []

    for q in questions:
        d, l = hybrid_retrieve(q, conn)
        all_dense.extend(d)
        all_lexical.extend(l)

    # Deduplicate by chunk id before merging
    seen_dense   = {}
    seen_lexical = {}
    for row in all_dense:
        seen_dense.setdefault(row[0], row)
    for row in all_lexical:
        seen_lexical.setdefault(row[0], row)

    return rrf_merge(
        list(seen_dense.values()),
        list(seen_lexical.values()),
    )


# ── Step 4: Reranker ───────────────────────────────────────────────
def rerank_step(data: dict) -> dict:
    question    = data["question"]
    candidates  = data["candidates"]
    top_chunks  = cohere_rerank(question, candidates, top_n=10)
    return {**data, "top_chunks": top_chunks}


# ── Step 5: Context + Prompt Assembly ─────────────────────────────
RAG_PROMPT = ChatPromptTemplate.from_template("""
You are a grounded question-answering assistant.

Answer ONLY using the supplied context.
Do NOT use external knowledge.
Cite chunk IDs for every claim (e.g., [Chunk 145]).
If the answer is not in the context, say:
"I don't have enough information to answer."

=== CONTEXT ===
{context}

=== QUESTION ===
{question}

=== ANSWER ===
""")
```

### 8.4 Full LCEL Chain

```python
def build_lcel_chain(conn):
    """
    Build and return the full LCEL hybrid RAG chain.
    Requires a live psycopg2 connection.
    """

    # Step 1: Rewrite query
    rewrite = RunnableLambda(
        lambda x: {"question": query_rewriter.invoke({"question": x["question"]}),
                   "original": x["question"]}
    )

    # Step 2: Expand into multiple queries
    expand = RunnableLambda(
        lambda x: {
            **x,
            "queries": [x["question"]] + multi_query_expander.invoke({"question": x["question"]})
        }
    )

    # Step 3: Retrieve candidates
    retrieve = RunnableLambda(
        lambda x: {
            **x,
            "candidates": multi_query_retrieve(x["queries"], conn)
        }
    )

    # Step 4: Rerank
    rerank = RunnableLambda(rerank_step)

    # Step 5: Build context string
    build_ctx = RunnableLambda(
        lambda x: {**x, "context": build_context(x["top_chunks"])}
    )

    # Step 6: Generate answer
    generate = RunnableLambda(
        lambda x: RAG_PROMPT.invoke({
            "question": x["original"],
            "context":  x["context"],
        })
    ) | llm | StrOutputParser()

    # ── Full chain ─────────────────────────────────────────────────
    chain = rewrite | expand | retrieve | rerank | build_ctx | generate

    return chain


# ── Conceptual summary ─────────────────────────────────────────────
#
# chain = (
#     query_rewriter
#     | multi_query_expander
#     | hybrid_retriever
#     | rrf_merger
#     | cohere_reranker
#     | context_builder
#     | rag_prompt
#     | llm
#     | StrOutputParser()
# )
```

### 8.5 Usage

```python
# ── Build and invoke ───────────────────────────────────────────────
conn  = psycopg2.connect(**DB_CONFIG)
chain = build_lcel_chain(conn)

result = chain.invoke({"question": "What is the capital of France?"})
print(result)

# ── Streaming (LCEL supports streaming natively) ───────────────────
for token in chain.stream({"question": "Explain the water cycle."}):
    print(token, end="", flush=True)

conn.close()
```

### 8.6 LCEL vs Pure Python — Comparison

| Feature | Pure Python | LangChain LCEL |
|---|---|---|
| **Streaming** | Manual | Built-in via `.stream()` |
| **Async** | Manual `asyncio` | Built-in via `.ainvoke()` |
| **Observability** | Manual logging | LangSmith integration |
| **Parallelism** | Manual threading | `RunnableParallel` |
| **Composability** | Function calls | `|` pipe operator |
| **Testability** | Easy unit tests | Easy mock injection |
| **Control** | Maximum | Slightly abstracted |
| **Debugging** | Straightforward | Requires LCEL knowledge |

---

## 9. Prompt Design & Grounded Generation

### 9.1 System Prompt Engineering for RAG

Key principles:

1. **Explicit grounding constraint** — "Answer ONLY from the context."
2. **Citation mandate** — "Cite chunk IDs for every claim."
3. **No-answer clause** — Exact fallback phrasing when context is insufficient.
4. **Temperature = 0** — Eliminates creative hallucination.

```python
SYSTEM_PROMPT = """You are a precise, grounded QA assistant.

Strict rules:
1. Use ONLY the context provided below to answer.
2. Do NOT draw on your training data or outside knowledge.
3. For every factual claim, cite supporting chunk IDs: [Chunk 145].
4. If the answer cannot be found in the context, respond exactly:
   "I don't have enough information to answer this question."
5. Never guess, infer, or extrapolate beyond what the context states.
6. Keep your answer concise and factually accurate."""
```

### 9.2 Context Formatting Best Practices

```python
def build_context_with_metadata(chunks: list[tuple],
                                 metadata: list[dict] = None) -> str:
    """
    Build rich context with optional source metadata.
    """
    parts = []
    for i, (chunk_id, chunk_text, score) in enumerate(chunks):
        meta_str = ""
        if metadata and i < len(metadata):
            m = metadata[i]
            src  = m.get("source", "unknown")
            page = m.get("page", "")
            meta_str = f"Source: {src}" + (f", Page: {page}" if page else "")

        block = f"[Chunk {chunk_id}]"
        if meta_str:
            block += f"  ({meta_str})"
        block += f"\n{chunk_text.strip()}"
        parts.append(block)

    return "\n\n---\n\n".join(parts)
```

### 9.3 Answer Quality Checks (Post-Generation)

```python
def validate_answer(answer: str, context: str) -> dict:
    """
    Basic heuristic checks on the generated answer.
    In production, use an LLM-as-judge or Cohere groundedness API.
    """
    no_info_phrase = "I don't have enough information"
    has_citations  = "[Chunk" in answer
    refused        = no_info_phrase in answer

    return {
        "has_citations":  has_citations,
        "refused":        refused,
        "needs_review":   not has_citations and not refused,
    }
```

---

## 10. Production Configuration & Sizing

### 10.1 Chunking Strategy

| Parameter | Recommended | Notes |
|---|---|---|
| Chunk size | 300–500 tokens | Larger = more context per chunk but less precise |
| Overlap | 15–20% | Prevents answer split across chunk boundary |
| Splitter | Token-aware | Character splitters can break mid-sentence |
| Hierarchy | Doc → Section → Chunk | Enables parent-doc retrieval if needed |

### 10.2 HNSW Tuning

```sql
-- Index-time parameters (set during CREATE INDEX)
-- m            = number of connections per layer (default 16)
-- ef_construction = candidate list size during construction (default 64)
-- Higher values → better recall, slower build, more memory

CREATE INDEX idx_chunks_embedding
ON document_chunks
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Query-time parameter
-- ef_search = candidate list size during search
-- Higher → better recall, slower query
-- Default: 40 — increase to 100–200 for production recall targets

SET hnsw.ef_search = 200;
```

### 10.3 PostgreSQL Connection Pooling

At 10M+ chunks, use **PgBouncer** or **pgpool-II**:

```
App Servers → PgBouncer (transaction mode) → PostgreSQL

Recommended PgBouncer settings:
  pool_mode           = transaction
  max_client_conn     = 1000
  default_pool_size   = 20
  min_pool_size       = 5
```

### 10.4 Scaling Milestones

| Corpus Size | Approach |
|---|---|
| < 1M chunks | Single PostgreSQL node, IVFFlat index |
| 1M–10M chunks | Single PostgreSQL node, HNSW index, PgBouncer |
| 10M–50M chunks | PostgreSQL read replicas for search workload |
| 50M+ chunks | pgvector on Citus (distributed), or dedicated vector DB (Weaviate, Qdrant) |

### 10.5 Cost Optimisation

| Stage | Optimisation |
|---|---|
| Embedding | Batch ingestion, cache frequent queries |
| Cohere Rerank | Only rerank top-N from RRF; Cohere charges per token |
| GPT-5 | Send only 5–10 chunks; temperature=0 for consistency |
| PostgreSQL | Partition `document_chunks` by `document_id` range for large corpora |

---
## 11. Common Misconceptions in Hybrid RAG

### 11.1 "PostgreSQL FTS is BM25"

**Misconception:** PostgreSQL's built-in full-text search (`ts_rank`) implements the BM25 ranking algorithm.

**Reality:** `ts_rank` is a custom scoring approximation — it does not implement BM25's term frequency saturation or document length normalisation. For true BM25, you need `pg_bm25` (ParadeDB) or an external engine like Elasticsearch. For most RAG workloads `ts_rank` is sufficient, but do not conflate the two when precision of ranking matters.

---

### 11.2 "RRF ranks are zero-indexed"

**Misconception:** Python's `enumerate()` can be used as-is to generate ranks for the RRF formula.

**Reality:** The RRF formula uses **1-based ranks**. `enumerate()` defaults to 0-based, which inflates the score of the top document (`1/(60+0) = 0.0167` instead of the correct `1/(60+1) = 0.0164`). Always pass `start=1`:

```python
# ❌ Wrong — 0-indexed
for rank, row in enumerate(dense_results):
    scores[doc_id] += 1 / (k + rank)

# ✅ Correct — 1-indexed as per Cormack et al., 2009
for rank, row in enumerate(dense_results, start=1):
    scores[doc_id] += 1.0 / (k + rank)
```

---

### 11.3 "The BM25 index needs a trigger like FTS"

**Misconception:** pg_bm25 (ParadeDB) requires a pre-computed column and a trigger, similar to the `search_vector TSVECTOR` pattern used in PostgreSQL FTS.

**Reality:** pg_bm25 builds its index **directly on the raw `chunk_text` column** using the Tantivy engine. There is no extra column, no trigger, and no `to_tsvector()` call involved. A normal `INSERT` is all that is needed — the index maintains itself like any standard B-tree index.

---

### 11.4 "Cohere returns documents by default"

**Misconception:** The Cohere rerank API always returns the document text alongside scores, so `return_documents` need not be specified.

**Reality:** Since Cohere API v2, only indices and relevance scores are returned by default. Document text must be explicitly requested:

```python
# ❌ Fragile — document text may not be returned
response = cohere_client.rerank(query=question, documents=texts, top_n=10, model="rerank-v3.5")

# ✅ Explicit and safe
response = cohere_client.rerank(query=question, documents=texts, top_n=10,
                                model="rerank-v3.5", return_documents=True)
```

---

### 11.5 "Passing embeddings as Python lists to psycopg2 is always safe"

**Misconception:** psycopg2 will correctly serialise a Python `list[float]` as a pgvector `vector` type without any special setup.

**Reality:** While this often works incidentally, the correct approach is to register the pgvector type adapter explicitly at connection time. Without it, behaviour can be inconsistent across driver versions:

```python
from pgvector.psycopg2 import register_vector

conn = psycopg2.connect(**DB_CONFIG)
register_vector(conn)   # ← ensures correct vector type serialisation
```

---

### 11.6 "Reranking replaces retrieval — just retrieve fewer candidates"

**Misconception:** Since Cohere reranking is so accurate, you can skip broad retrieval and just fetch 10–20 candidates directly.

**Reality:** The reranker is a **precision** tool, not a **recall** tool. It can only re-order what it is given — it cannot surface documents that were never retrieved in the first place. If you retrieve too few candidates, high-quality chunks may never reach the reranker. The correct approach is always broad retrieval (100+ per retriever) followed by narrow reranking.

---

### 11.7 "Temperature > 0 adds helpful creativity in RAG"

**Misconception:** A small temperature (e.g. 0.3–0.7) makes the LLM's answers more natural and readable without meaningfully increasing hallucination.

**Reality:** In a grounded RAG system, **any temperature above 0 increases the probability of the model deviating from the retrieved context**. The model may blend retrieved facts with parametric memory in unpredictable ways. For production QA over a closed corpus, always use `temperature=0`.

---

## 12. Quick Reference Cheat Sheet

```
┌────────────────────────────────────────────────────────────────┐
│              HYBRID RAG — QUICK REFERENCE                      │
├────────────────────────────────────────────────────────────────┤
│ CHUNKING                                                       │
│   Size:     300–500 tokens                                     │
│   Overlap:  15–20%                                             │
│   Method:   Token-aware sliding window                         │
├────────────────────────────────────────────────────────────────┤
│ EMBEDDING                                                      │
│   Model:    text-embedding-3-large                             │
│   Dim:      3072                                               │
│   Op:       cosine similarity (<=>)                            │
├────────────────────────────────────────────────────────────────┤
│ PGVECTOR INDEX                                                 │
│   Type:     HNSW                                               │
│   Params:   m=16, ef_construction=64                           │
│   Runtime:  SET hnsw.ef_search = 200;                          │
├────────────────────────────────────────────────────────────────┤
│ FTS INDEX                                                      │
│   Type:     GIN on tsvector                                    │
│   Function: plainto_tsquery / ts_rank                          │
│   True BM25: pg_bm25 / ParadeDB (optional)                     │
├────────────────────────────────────────────────────────────────┤
│ RETRIEVAL SIZES                                                │
│   Dense:     top-100                                           │
│   Lexical:   top-100                                           │
│   RRF output: top 100–150                                      │
│   Cohere out: top 10                                           │
│   To LLM:    top 5–10                                          │
├────────────────────────────────────────────────────────────────┤
│ RRF                                                            │
│   Formula:  1 / (k + rank_i)   summed over retrievers          │
│   k:        60 (Cormack default)                               │
│   Ranks:    1-indexed                                          │
├────────────────────────────────────────────────────────────────┤
│ COHERE RERANK                                                  │
│   Model:    rerank-v3.5                                        │
│   Type:     Cross-encoder (query + doc together)               │
│   Input:    100–150 candidates                                 │
│   Output:   top-10                                             │
├────────────────────────────────────────────────────────────────┤
│ LLM GENERATION                                                 │
│   Model:    GPT-5 (gpt-5 / verify model string)                │
│   Temp:     0 (deterministic)                                  │
│   Prompt:   Context-only, cite chunk IDs                       │
│   Fallback: "I don't have enough information."                 │
├────────────────────────────────────────────────────────────────┤
│ PIPELINE ORDER                                                 │
│   Question → Rewrite → Multi-Query → Dense+Lexical             │
│   → RRF → Cohere Rerank → Context Build → GPT-5 → Answer       │
└────────────────────────────────────────────────────────────────┘
```

