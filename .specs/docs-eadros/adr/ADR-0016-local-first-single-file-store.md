# ADR-0016: One SQLite file is the whole store — no server, no sidecar

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-07-29 |
| **Amends** | [ADR-0010 — Knowledge base & vector index](ADR-0010-knowledge-base.md) (the KB stays; the dedicated vector store does not) |
| **Related** | [`architecture/CONTAINERS.md`](../architecture/CONTAINERS.md) · [`architecture/DATA_MODEL.md`](../architecture/DATA_MODEL.md) · [ADR-0005 — Provider abstraction](ADR-0005-provider-abstraction.md) |

## Context

The architecture as specified does not describe one runtime. `CONTAINERS.md` declares a
**Node.js/TypeScript** CLI, then lists **ChromaDB** as a *"local embedded vector database"* — there
is no embedded ChromaDB for JavaScript; the JS client talks to a server — alongside **Redis** for
the event bus. ADR-0005 names **LiteLLM**, which is Python. `DATA_MODEL.md` is written in
**PostgreSQL** (`gen_random_uuid()`, `TIMESTAMP WITH TIME ZONE`).

Assembled honestly, the default install is: a Node CLI, a Postgres server, a Redis server, a Chroma
server, and a Python runtime for the provider proxy. Five processes and two language ecosystems for
a tool whose stated user is **a solo open-source maintainer with no time** — the same person the
vision says is too busy to write a LinkedIn post. The install cost exceeds the problem it solves,
and this is the most common way a good open-source tool gets tried once and abandoned.

The scale does not justify any of it. One repository produces perhaps a few thousand rows a year;
its knowledge base is a few hundred documents — ADRs, past posts, the README. These are numbers a
laptop handles without a server, and a vector database sized for millions of embeddings is
answering a question nobody asked.

Two requirements do constrain the choice, and they pull the same way. The **outbox** must commit
intent and outcome atomically or `publish` can double-post ([`publish.md`](../commands/publish.md)),
and the **transition ledger** must be written in the same transaction as the state change it
records, or the audit trail can disagree with reality. Both need real transactions.

## Options considered

**A. A single SQLite file holding everything — state, content, outbox, metrics, the KB** *(chosen)*

- ✅ **Zero-install is the adoption argument.** `npx eadros weekly` runs with no Docker, no
  services, no Python. For the stated user this is not convenience; it is the difference between
  being used and being starred.
- ✅ **Real transactions, which two core mechanisms require.** The outbox commits intent before the
  platform call and the outcome after it, in the same store as the state machine. Flat files cannot
  do this, and the failure mode when they cannot is a public double post.
- ✅ **The KB fits without a vector database.** SQLite **FTS5** for lexical search plus a small
  embeddings table; at a few thousand vectors an exhaustive cosine scan completes in microseconds.
  ADR-0010's *purpose* — accurate, non-repetitive, brand-aligned content — is met; only its
  implementation choice changes.
- ✅ **One runtime.** Dropping Chroma and LiteLLM from the default path removes the Python
  ecosystem entirely. ADR-0005's provider abstraction is served by native HTTP clients, which is
  what it needed anyway — LiteLLM was an implementation convenience, not a requirement.
- ✅ **The whole program state is one file** — copyable, diffable in CI, attachable to a bug report,
  deletable to start over. For a governance tool the auditability of "here is the file" is worth
  more than query sophistication.
- ❌ **Single-writer concurrency.** Accepted and stated: this is a per-repository CLI, not a
  multi-tenant service. WAL mode handles concurrent readers; concurrent writers are prevented by
  the same `manifest_rev` optimistic-concurrency counter the manifest already uses.
- ❌ **A future multi-repository workspace will need a migration.** Accepted — see option C.

**B. Postgres + Redis + Chroma (the currently implied stack)** — ✅ scales, familiar, correct for a
hosted product; ❌ imposes an operations burden on a user who has none of the scale that justifies
it, and contradicts the vision's own description of who this is for. **Rejected.**

**C. A storage abstraction layer from day one, SQLite or Postgres** — ✅ no migration later, looks
prudent; ❌ premature. The access patterns are not known yet, and an abstraction written before them
constrains the schema to the intersection of two engines while the product is still learning what it
needs. **The abstraction costs more now than the migration will cost later.** Deferred until the V2
multi-repository workspace exists and its access patterns are real; until then the schema simply
avoids SQLite-only constructs where the cost of avoiding them is zero.

**D. Flat files (JSON/YAML) on disk** — ✅ maximally inspectable, no dependency at all, appealing
for a docs-adjacent tool; ❌ **no transactions**, so the outbox cannot be made safe and the ledger
can drift from the state it describes. Rejected on the one requirement that has no workaround.

## Decision

1. **One SQLite database file** (`.eadros/eadros.db`), WAL mode, `STRICT` tables, holding pipeline
   state, content, approvals, the outbox, the metrics time series, the event log and the knowledge
   base.
2. **In-process event bus**, with every event **persisted to the same file in the same transaction**
   as the state change it accompanies. Redis becomes an optional adapter for a future hosted mode,
   not a default dependency.
3. **KB via FTS5 + an embeddings table.** No dedicated vector store in the default path.
4. **No Python in the default path.** Provider integrations are native HTTP against each vendor's
   API; Ollama is a local HTTP endpoint like any other.
5. **Postgres is a V2 concern**, gated on the multi-repository workspace actually existing. The
   schema carries a `repo_id` from the start so that migration is a data move rather than a redesign.

## Consequences

- `CONTAINERS.md` must be rewritten: the container diagram currently describes a system this ADR
  replaces, and leaving both in the repository would leave a reader unable to tell which is true.
- ADR-0010's vector-index decision is superseded in mechanism while its intent is preserved — worth
  stating plainly, because "we dropped the vector database" reads as a regression unless the scale
  argument travels with it.
- Backup, reset and support all become file operations, which is the correct affordance for a tool
  a single person runs.
- The single-writer constraint is a documented limit rather than a discovered one; `/eadros status`
  reports the lock holder when a second process tries to write.
- A hosted multi-repository EADROS is a genuinely different product with a genuinely different
  store. Saying so now is cheaper than pretending one schema serves both.
