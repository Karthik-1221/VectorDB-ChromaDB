# 📄 ChromaDB One-Page Cheatsheet

## Install
```bash
pip install chromadb sentence-transformers
```

## Client Types
```python
import chromadb

chromadb.Client()                                  # in-memory, ephemeral
chromadb.PersistentClient(path="./chroma_db")       # local disk
chromadb.HttpClient(host="localhost", port=8000)    # remote server
```

## Collections
```python
client.create_collection("name")            # errors if exists
client.get_collection("name")                # errors if missing
client.get_or_create_collection("name")      # safe — use this by default
client.delete_collection("name")
client.list_collections()
```

## Insert / Update
```python
collection.add(ids=[...], documents=[...], metadatas=[...], embeddings=[...])
# ↑ errors if id already exists

collection.upsert(ids=[...], documents=[...], metadatas=[...], embeddings=[...])
# ↑ insert or update — SAFE for re-runs, use in production pipelines

collection.update(ids=[...], documents=[...], metadatas=[...])
```
> `embeddings=` is optional — if omitted, Chroma auto-embeds `documents` using the collection's embedding function.

## Query (Similarity Search)
```python
collection.query(
    query_texts=["search string"],      # OR query_embeddings=[[...]]
    n_results=5,
    where={"category": "billing"},      # metadata filter
    where_document={"$contains": "error"},  # full-text filter
    include=["documents", "distances", "metadatas"]
)
```

## Metadata Filter Operators
```python
{"category": "billing"}                          # exact match
{"price": {"$gt": 100}}                           # $gt $gte $lt $lte $ne $eq
{"$and": [{"status": "open"}, {"priority": "high"}]}
{"$or": [{"category": "bug"}, {"category": "crash"}]}
```

## Retrieve / Delete
```python
collection.get(ids=["id1"])
collection.get(where={"category": "billing"})
collection.peek()                    # first 10 items — quick inspection
collection.count()

collection.delete(ids=["id1"])
collection.delete(where={"category": "deprecated"})
```

## Similarity → Distance Conversion (cosine)
```python
similarity = 1 - distance   # cosine distance returned by query()
```

## Common Embedding Models
| Model | Dims | Notes |
|-------|------|-------|
| `all-MiniLM-L6-v2` | 384 | Fast, free, local — best for prototyping |
| `bge-large-en` | 1024 | Higher quality, slower |
| OpenAI `text-embedding-3-small` | 1536 | API-based, strong general quality |

## Golden Rules
- ✅ Use `upsert()` in production, not `add()`
- ✅ Use `get_or_create_collection()`, not `create_collection()`
- ✅ Never mix embeddings from two different models in one collection
- ✅ Always match embedding model between ingestion and query time
- ⚠️ `where` filters use **exact key/value match** — casing matters
