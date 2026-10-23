# 05 — RAG and Retrieval, 2026 Edition

[← 04 MCP](04-mcp-and-protocols.md) · [Index](00-INDEX.md) · Next: [06 FastAPI →](06-fastapi-fundamentals.md)

---

## 1. The mental model

**RAG (Retrieval-Augmented Generation)** = find relevant information, put it in the prompt, have the model answer grounded in it (ideally with citations).

```
INGEST:  docs → parse → chunk → embed → store (vector + keyword index + metadata)
QUERY:   question → (rewrite) → retrieve top-k → rerank → top-n into prompt → answer + citations
```

For a CV engineer: retrieval is **metric learning + nearest-neighbor search**, the same idea as image retrieval with embeddings and FAISS.

---

## 2. Each stage, and what seniors get right

### Parsing (where most quality is lost)
- PDFs, slides, scanned docs and tables are messy. Layout-aware parsing (OCR, table extraction, or a VLM reading page images) often matters more than the choice of embedding model.
- **This is a CV problem.** Document AI (layout detection, table structure, OCR, VLM-based parsing) is a strong niche for you.

### Chunking
- Fixed-size (for example 500–1,000 tokens with overlap) is a decent baseline.
- Better: structure-aware (split on headings and sections), keep tables whole, attach the parent section title.
- **Contextual retrieval**: prepend a short LLM-generated summary of where the chunk sits in the document before embedding. Cheap and effective.
- Parent-child: retrieve small chunks, feed the larger parent section to the model.

### Embedding and indexing
- Embedding models: hosted (OpenAI, Voyage, Cohere, Google) or open (BGE, E5, GTE, Nomic, Jina families). Pick using your own eval, not leaderboard rank alone.
- Index: HNSW is the default ANN algorithm. Stores: **pgvector** (Postgres; great default if you already run Postgres), Qdrant, Weaviate, Milvus, Pinecone, Elasticsearch/OpenSearch, LanceDB, Chroma for prototypes.
- Store **metadata** (source, date, tenant, permissions) and filter on it. This is how you do multi-tenant security.

### Hybrid search (do this by default)
Dense vectors miss exact terms (product codes, names, error IDs). **BM25 keyword search** catches them. Combine both with **Reciprocal Rank Fusion (RRF)**:

```python
def rrf(rankings: list[list[str]], k: int = 60) -> list[str]:
    scores: dict[str, float] = {}
    for ranking in rankings:
        for rank, doc_id in enumerate(ranking):
            scores[doc_id] = scores.get(doc_id, 0.0) + 1.0 / (k + rank + 1)
    return sorted(scores, key=scores.get, reverse=True)
```

### Reranking
Retrieve 50–100 candidates cheaply, then rerank with a **cross-encoder** (Cohere Rerank, BGE-reranker, or an LLM) to the top 5–10. Usually the single biggest quality gain after fixing parsing.

### Generation
- Instruct the model to answer **only** from sources, cite chunk IDs, and say "I don't know" when the sources don't cover it.
- Put the most relevant chunks first or last. Models attend less to the middle of long contexts.

---

## 3. Agentic RAG

Classic RAG retrieves once. **Agentic RAG** gives the model retrieval as a *tool* and lets it search repeatedly: reformulate, follow references, combine sources.

- Query rewriting and decomposition ("compare policy A and B" becomes two searches).
- Iterative search until the evidence is sufficient.
- Routing between sources (vector DB, SQL, web, an API).
- **Text-to-SQL** for structured data, often better than embedding tables.
- **GraphRAG / knowledge graphs** for questions about relationships across many documents. Powerful, but costly to build, so use it only when relationships really matter.

Coding agents mostly use **grep + file reading** instead of vector search. For code and well-structured corpora, agentic keyword search is often enough.

---

## 4. Long context vs RAG

With 1M-token windows, "just put it all in the prompt" sometimes works.

| Use long context when | Use RAG when |
|---|---|
| Corpus is small (< a few hundred pages) and fixed per request | Corpus is large or growing |
| You need whole-document reasoning | You need per-user permissions on documents |
| Prompt caching makes repeated use cheap | Latency and cost per query matter |
| | You need citations and freshness |

Often the answer is both: retrieve the relevant *documents*, then give the model whole documents rather than tiny chunks.

---

## 5. Evaluating RAG

Measure retrieval and generation **separately**:
- **Retrieval**: recall@k, MRR, nDCG on a labeled set of question → relevant-chunk pairs.
- **Generation**: faithfulness/groundedness (is every claim supported by the sources?), answer relevance, correctness against reference answers.
- Tools: Ragas, DeepEval, or your own LLM-as-judge with a rubric ([09](09-evals-observability.md)).
- Build 50–200 real questions from users or domain experts. That beats any synthetic benchmark.

---

## 6. Check yourself

- Why does hybrid search beat pure vector search for enterprise data?
- Where do you enforce document permissions in a multi-tenant RAG system?
- Your RAG system gives wrong answers. How do you work out whether retrieval or generation is at fault?
- When would you choose GraphRAG? When text-to-SQL?
- Explain reranking and why it's a separate step.
