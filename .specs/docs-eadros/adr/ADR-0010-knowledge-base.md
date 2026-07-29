# ADR-0010: Project Knowledge Base & Vector Index

* **Status:** Approved — **amended by [ADR-0016](ADR-0016-local-first-single-file-store.md)**. The knowledge base stands; the dedicated vector store does not. At a few hundred documents, SQLite FTS5 plus an embeddings table meets the same intent without a server.
* **Date:** 2026-07-28
* **Context:** Agents need context on the project's vision, architecture, competitor positioning, and past posts to avoid repetitive content.
* **Decision:** EADROS maintains a local Vector Store (`eadros_kb`) indexing the repository's README, ADRs, RFCs, docs, and previous post history.
* **Consequences:** Ensures generated posts are accurate, brand-aligned, and non-repetitive.
