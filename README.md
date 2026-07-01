# 🧭 Vector Databases & ChromaDB — Interview & Project Prep

> A concise, practical guide to vector databases for junior backend/data engineers — grounded in ChromaDB, aimed at interviews and real projects.

## 📋 Table of Contents

1. [When Vector DBs vs Relational DBs](#1-when-vector-dbs-vs-relational-dbs)
2. [Vector DB Primer](#2-vector-db-primer)
3. [ChromaDB Specifics](#3-chromadb-specifics)
4. [Example Workflow](#4-example-workflow)
5. [Practical Tips & Pitfalls](#5-practical-tips--pitfalls)
6. [Interview Checklist](#6-interview-checklist)

---

## 1. When Vector DBs vs Relational DBs

Relational databases excel at **exact-match queries** — `WHERE user_id = 42` has one right answer. Vector databases exist for a fundamentally different problem: **semantic similarity search**, where you want "find things *like* this," not "find things *equal to* this." A vector DB stores embeddings — numerical representations of meaning — and finds nearest neighbours in that space. **ChromaDB** is relevant because it's the lowest-friction entry point: it's open-source, runs in-process with zero infrastructure (`pip install chromadb`), embeds documents automatically, and scales from a prototype notebook to a small production RAG system without changing your code — making it the standard first vector DB most engineers learn.

---

## 2. Vector DB Primer

- **Vectors/Embeddings** — arrays of floats (e.g., 384 or 1536 dimensions) produced by an embedding model, representing the *meaning* of text/images so similar content lands close together in vector space.
- **Similarity metrics:**
  - **Cosine similarity** — measures the angle between vectors (ignores magnitude); the default for most text embeddings.
  - **Dot product** — like cosine but sensitive to magnitude; faster, used when vectors are pre-normalised.
  - **L2 (Euclidean distance)** — straight-line distance; common for image embeddings.
- **ANN (Approximate Nearest Neighbour)** — exact nearest-neighbour search is too slow at scale (O(n) per query), so vector DBs use approximate algorithms that trade a small amount of accuracy for massive speed gains.
- **Common index types:**
  - **HNSW** (Hierarchical Navigable Small World) — graph-based, fast queries, high recall; what ChromaDB uses by default.
  - **IVF** (Inverted File Index) — clusters vectors, searches only relevant clusters; good for very large datasets.
  - **PQ** (Product Quantization) — compresses vectors to save memory; often combined with IVF.
- **Dimensionality trade-offs** — higher dimensions capture more nuance but increase memory, storage, and query latency; 384–768 dims is a common practical sweet spot.
- **Recall vs latency** — higher recall (finding the *true* nearest neighbours) generally costs more latency; index parameters (like HNSW's `ef_search`) let you tune this trade-off.

---

## 3. ChromaDB Specifics

**Architecture:** Chroma stores data in **collections** (like a table). Each collection holds documents, their embeddings, metadata, and IDs, indexed internally with HNSW for fast similarity search.

**Main features:** automatic embedding generation, metadata filtering, persistence to disk, three client modes, and a simple Pythonic API.

**Client modes:**

| Client | Use Case |
|--------|----------|
| `chromadb.Client()` | In-memory, data lost on exit — quick prototyping |
| `chromadb.PersistentClient(path=...)` | Local disk storage — development & small production |
| `chromadb.HttpClient(host=..., port=...)` | Connects to a running Chroma server — multi-service production |

**Typical API patterns:**

```python
import chromadb

client = chromadb.PersistentClient(path="./chroma_db")

collection = client.get_or_create_collection("my_collection")  # create/reuse

collection.upsert(ids=["id1"], documents=["text"], metadatas=[{"category": "a"}])
collection.query(query_texts=["search text"], n_results=5, where={"category": "a"})
collection.get(ids=["id1"])          # retrieve by ID
collection.delete(ids=["id1"])       # delete by ID
collection.delete(where={"category": "old"})  # delete by filter
```

---

## 4. Example Workflow

```bash
pip install chromadb sentence-transformers
```

```python
import chromadb
from sentence_transformers import SentenceTransformer

# 1. Set up a persistent client and an embedding model
client = chromadb.PersistentClient(path="./chroma_db")
embedder = SentenceTransformer("all-MiniLM-L6-v2")  # 384-dim, fast, free, local

# 2. Create (or reuse) a collection
collection = client.get_or_create_collection("support_tickets")

# 3. Upsert documents with metadata (embeddings computed manually here,
#    but Chroma can also auto-embed if you skip the `embeddings=` arg)
docs = [
    "My invoice shows the wrong billing amount",
    "The app crashes when I upload a large file",
    "How do I reset my password?",
]
metadatas = [
    {"category": "billing", "priority": "high"},
    {"category": "bug", "priority": "high"},
    {"category": "account", "priority": "low"},
]
ids = ["t1", "t2", "t3"]

embeddings = embedder.encode(docs).tolist()  # convert numpy -> plain list

collection.upsert(ids=ids, documents=docs, metadatas=metadatas, embeddings=embeddings)

# 4. Similarity search with a metadata filter
query = "I was charged the wrong amount"
query_embedding = embedder.encode([query]).tolist()

results = collection.query(
    query_embeddings=query_embedding,
    n_results=2,
    where={"category": "billing"}       # only search billing tickets
)

# 5. Retrieve and print results
for doc, dist in zip(results["documents"][0], results["distances"][0]):
    print(f"[{1 - dist:.3f} similarity] {doc}")
```

---

## 5. Practical Tips & Pitfalls

- **Choosing embedding models** — `all-MiniLM-L6-v2` (384-dim) is fast and free for prototyping; upgrade to larger models (OpenAI `text-embedding-3-small`, or `bge-large`) when recall quality matters more than speed.
- **Hybrid search** — pure vector search misses exact keyword matches (e.g., product SKUs, error codes). Combine vector search with keyword search (BM25) and merge results for better coverage.
- **Scaling strategies** — start with `PersistentClient` locally; move to `HttpClient` + a dedicated Chroma server (or Chroma Cloud) once multiple services need shared access or dataset size grows beyond single-machine comfort.
- **Persistence/backups** — `PersistentClient` writes to disk automatically, but back up the storage directory like any database; there's no built-in replication for self-hosted instances.
- **Data versioning** — track which embedding model version generated your vectors; re-embedding with a new model without re-indexing old data silently breaks similarity search.
- **Embedding drift** — if you swap embedding models later, old and new vectors are **not comparable** — you must re-embed the entire collection, not just new documents.
- **Metadata mismatches** — filtering with `where={"category": "billing"}` silently returns zero results if the key or value casing doesn't exactly match what was stored — validate metadata schemas consistently at ingestion time.

---

## 6. Interview Checklist

**Common questions to expect:**
- Explain the difference between exact and approximate nearest-neighbour search, and why ANN is necessary at scale.
- Compare cosine similarity vs L2 distance — when would you choose one over the other?
- What happens if you query a collection with an embedding from a different model than what indexed it?
- How would you design metadata filtering for a multi-tenant application?
- Describe how HNSW works at a high level, and its recall/latency trade-off.

👉 See **Deliverable C** below for 6 full practice problems with solution hints.

---

<div align="center">

Made with 🧭 by [Karthik Boodidha](https://www.linkedin.com/in/karthik-boodidha)

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/karthik-boodidha)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/Karthik-1221)

</div>
