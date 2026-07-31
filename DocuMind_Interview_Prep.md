# DocuMind — Interview Prep Notes

## What it does
A full-stack RAG (Retrieval-Augmented Generation) platform: users upload documents, then search and chat with them via an LLM. Frontend in Next.js, backend in FastAPI.

## How it works

**1. Ingestion**
Uploaded docs are chunked and embedded, then stored for retrieval — text chunks in Postgres (via `pgvector`) for semantic search, and indexed for BM25 keyword search.

**2. Hybrid Search**
Combines two retrieval methods:
- **pgvector (dense/semantic)** — embeds the query, finds chunks with closest vector similarity. Good at catching meaning/paraphrase.
- **BM25 (sparse/keyword)** — classic TF-IDF-style ranking. Good at exact term/entity matches (names, codes, jargon) that embeddings can blur.
- Results from both are merged/re-ranked (e.g. reciprocal rank fusion) to improve recall and accuracy over either alone.

**3. Response generation**
Retrieved chunks are passed to an LLM as context to answer the user's query. Planned: stream the answer back token-by-token via **Server-Sent Events (SSE)** instead of waiting for the full response — cuts perceived latency.

**4. Auth & security**
- **Google OAuth** for login → issues a **JWT**.
- JWT stored in an **HttpOnly cookie** (not accessible to JS) → mitigates **XSS** token theft.
- Needs CSRF protection alongside this (e.g. SameSite cookie attribute + CSRF token) since cookies auto-attach to requests.

**5. Multi-tenancy**
**Tenant-level data isolation** — retrieval queries are scoped so one user/org never sees another's documents (e.g. filtering vector search by tenant ID, row-level security in Postgres).

## Core concepts to know cold

| Concept | Be ready to explain |
|---|---|
| RAG | Why retrieval beats pure LLM knowledge (freshness, grounding, reduces hallucination) |
| Embeddings / vector search | How text becomes vectors, cosine similarity, why pgvector uses approximate nearest-neighbor indexes (e.g. HNSW/IVFFlat) for speed at scale |
| BM25 | TF-IDF intuition, term frequency + inverse doc frequency, why it beats embeddings on exact matches |
| Hybrid retrieval | Why combine dense + sparse; fusion/re-ranking strategies |
| SSE vs WebSockets | SSE is one-way (server→client), simpler, works over HTTP; WebSockets are bidirectional — justify why SSE fits a "stream an answer" use case |
| JWT vs sessions | Stateless vs stateful auth, tradeoffs |
| HttpOnly cookies | Prevents JS (and thus XSS payloads) from reading the token |
| CSRF | Why it's a separate risk from XSS even with HttpOnly cookies, and how SameSite/CSRF tokens address it |
| Multi-tenancy patterns | Row-level security vs separate schemas vs separate DBs — tradeoffs at your chosen approach |

## Likely interview questions
- "Why hybrid search instead of just embeddings?" → keyword precision on names/codes/rare terms that embeddings miss.
- "How do you prevent tenant data leaking in the RAG pipeline?" → filter at the retrieval query level, not just the UI.
- "Why SSE over polling or WebSockets here?" → one-directional streaming need, simpler infra, HTTP-native.
- "Walk me through what happens when a user uploads a doc and asks a question." → ingestion → chunk/embed → dual retrieval → fusion → LLM context → streamed answer.
- "What happens if BM25 and vector search disagree?" → talk about your fusion/re-ranking approach.

## Gaps to shore up before the interview
Since auth and streaming are marked as in-progress, be ready to speak to design *intent* even if implementation isn't finished — interviewers often probe unfinished pieces to test understanding, not completion.
