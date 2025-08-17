# Qdrant Setup — AutoSpec Architect

This file documents the **entire vector database setup** used by the app’s RAG pipeline.
It explains the collection, payload schema, indexes, filters, env vars, and how we use it.

---

## 1) Purpose

Qdrant is the **vector DB** that stores embeddings of PRD chunks.  
The backend queries Qdrant to retrieve the most relevant chunks (by similarity + filters) and feeds them to the LLM.

- **Store:** one point per PRD chunk (dense vector + metadata payload)
- **Search:** semantic similarity (Cosine) + filters (`user_id`, `upload_id`, etc.)
- **Scope:** single collection shared by all users; multi-tenancy enforced via payload filters

---

## 2) Cluster & Keys (Cloud)

- **Cluster:** Qdrant Cloud (region near backend)
- **API Key:** Read + Write (server-only)
- **Endpoint (HTTPS):** cluster public URL

These are exposed to the backend via env vars (see §7).

---

## 3) Collection

**Name:** `prd_chunks_v1`  
**Use case:** Global search (single collection for all PRDs)  
**Vectors:** one dense vector per point  
**Vector size (dim):** `1536` (assumes OpenAI `text-embedding-3-small` or equivalent)  
**Distance metric:** `Cosine`  
**Shards/Replicas:** 1/1 (dev defaults; increase in prod)

> If you change embedding model/dimension later, create a **new versioned collection** (e.g., `prd_chunks_v2`) and migrate or re-embed.

---

## 4) Payload (Metadata) Schema

Each Qdrant **point** = one PRD chunk.

| Field        | Type     | Required | Notes                                                     |
|--------------|----------|----------|-----------------------------------------------------------|
| `user_id`    | keyword  | yes      | Owner; used for per-user isolation                        |
| `upload_id`  | keyword  | yes      | ID of the PRD (maps to `uploads.id` in Supabase)          |
| `chunk_id`   | keyword  | no       | Unique chunk identifier (optional but useful)             |
| `page`       | integer  | no       | Page number                                               |
| `order`      | integer  | yes      | Chunk order within the document                           |
| `mime`       | keyword  | no       | Source MIME, e.g., `application/pdf`                      |
| `hash`       | keyword  | no       | Content hash of the chunk                                 |
| `text`       | text     | no       | Raw text (optional; OK to truncate/omit for privacy)      |
| `created_at` | datetime | no       | ISO timestamp                                             |

**Indexes (created):**
- `user_id` → **keyword**
- `upload_id` → **keyword**
- `page` → **integer**
- `order` → **integer**

These support fast filtering (multi-tenant + per-document).

---

## 5) Query Pattern (how backend will use it)

- **Insert/Upsert:** one point per chunk with vector + payload above.
- **Search (Top-K):** query vector + filters:
  - `must: [{ key: "user_id", match: { value: <uid> } }, { key: "upload_id", match: { value: <prd_id> } }]`
  - optional: sort by `order`, filter by `page`, etc.
- **Typical limits:** `top_k = 4..12` chunks per query to build the LLM context.
- **Multi-tenancy:** enforced entirely via the `user_id` filter.

---

## 6) Operational Guidance

- **Versioning:** bump collection name when changing embedding size/model (`prd_chunks_v2`).
- **Dimension mismatch:** Qdrant will reject inserts if vector length ≠ configured size.
- **Throughput:** increase shards/replicas and `write_consistency: "majority"` in prod.
- **Latency tuning:** raise search `ef` for higher recall (at some latency cost).
- **Data lifecycle:** delete all points for an `upload_id` when a PRD is removed.

---

## 7) Environment Variables (backend)

Set these in the backend `.env` (local and on Render):

```ini
QDRANT_URL=<https endpoint of your cluster>
QDRANT_API_KEY=<qdrant api key>
QDRANT_COLLECTION=prd_chunks_v1
