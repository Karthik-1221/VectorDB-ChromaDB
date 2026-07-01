# 🎯 Vector Database & ChromaDB — Practice Questions

Six practice problems mixing system design and short coding tasks, with solution hints (not full solutions — work through them yourself first).

---

### Q1 — Design: Multi-tenant document search

**Problem:** Design a ChromaDB-backed system where multiple companies (tenants) each have their own private set of documents, but all share the same infrastructure.

**Hint:** Two approaches — (a) one collection per tenant (`collection_{tenant_id}`), simple isolation but many collections to manage; (b) one shared collection with a mandatory `tenant_id` metadata field, filtered on every single query via `where={"tenant_id": ...}`. For interviews, discuss the trade-off: (b) is easier to manage at scale but requires **strict enforcement** — a missing filter is a data leak. Mention this as a security-critical detail.

---

### Q2 — Coding: Handle re-ingestion safely

**Problem:** Write a function `ingest_documents(collection, docs, ids, metadatas)` that can be run repeatedly (e.g., on a daily cron job) on the same source documents without creating duplicates or erroring out.

**Hint:** Use `collection.upsert()`, not `add()` — `add()` throws on duplicate IDs, `upsert()` handles insert-or-update cleanly. Also think about **stable ID generation** — hash the document content or use a consistent source-derived ID (e.g., `f"{filename}_{chunk_index}"`) so the same document always maps to the same ID across runs.

---

### Q3 — Design: Recall vs latency trade-off

**Problem:** Your ChromaDB-backed search is returning results in 5ms but users complain the results aren't relevant enough. How do you diagnose and fix this?

**Hint:** This is likely a **recall problem**, not a latency problem — HNSW's approximate search may be missing true nearest neighbours. Discuss: (1) increasing `n_results` and re-ranking downstream, (2) checking if the embedding model itself is a poor fit for the domain (e.g., using a generic model for highly technical text), (3) verifying query and document embeddings were generated with the *same* model. Also mention hybrid search (vector + keyword) as a mitigation.

---

### Q4 — Coding: Metadata filter bug

**Problem:** The following query returns zero results even though matching documents exist:

```python
collection.query(query_texts=["refund status"], where={"Category": "billing"})
```

Documents were inserted with `metadatas=[{"category": "billing"}, ...]`. Find and explain the bug.

**Hint:** Metadata filtering in Chroma is **case-sensitive and exact-match** — `"Category"` (capital C) does not match the stored `"category"` (lowercase). This is a classic "silent failure" bug: no error is thrown, it just returns an empty result set. Always validate metadata key casing consistency at ingestion time, ideally with a shared constants file or schema.

---

### Q5 — Design: Embedding model migration

**Problem:** You've been running a production ChromaDB collection with `all-MiniLM-L6-v2` (384-dim) for six months. You want to upgrade to a better 1024-dim model. What's your migration plan?

**Hint:** You **cannot** mix old and new embeddings in the same collection — different models produce vector spaces that aren't comparable, even if dimensions matched. Plan: (1) create a new collection, (2) re-embed all existing documents with the new model and upsert into the new collection, (3) run both collections in parallel temporarily for validation/A-B testing, (4) cut over reads to the new collection, (5) deprecate the old one. Emphasize this is a **full re-index**, not an incremental migration — mention the compute/time cost this implies for large datasets.

---

### Q6 — Coding: Build a simple hybrid search function

**Problem:** Write a function that combines ChromaDB vector search results with a simple keyword-match filter, returning only documents that satisfy both.

**Hint:** Start with `collection.query()` to get top-k vector matches, then apply a Python-side keyword filter (`if keyword.lower() in doc.lower()`) on the returned documents — or use Chroma's built-in `where_document={"$contains": "keyword"}` to push the keyword filter into the query itself. Discuss the trade-off: filtering client-side after retrieval risks missing relevant docs that fell outside `n_results`, so widening `n_results` before filtering (or using `where_document` directly) is usually safer.

```python
def hybrid_search(collection, query, keyword, n_results=10):
    results = collection.query(
        query_texts=[query],
        n_results=n_results,
        where_document={"$contains": keyword}
    )
    return results
```

---

## 💡 General Interview Tips

- Always mention **trade-offs**, not just the "right" answer — interviewers are testing judgment, not memorisation
- If asked to design something, sketch the schema (collection structure, metadata fields) before writing code
- Know the **failure modes** cold: embedding drift, metadata mismatches, mixing models — these come up constantly in real interviews because they're the actual pain points engineers hit in production
