# EADROS Architecture: Containers (C4 Level 2)

**One process, one file, no services.** The default install is a single CLI binary and a SQLite
database ([ADR-0016](../adr/ADR-0016-local-first-single-file-store.md)). There is no Postgres, no
Redis, no vector-database server and no Python runtime in the default path — a tool whose stated
user is a maintainer with no time cannot ask for five processes before it does anything.

```
┌─ EADROS process (Node.js / TypeScript) ───────────────────────────────┐
│                                                                        │
│  [CLI & review surface]                                                │
│         │ commands                                                     │
│         ▼                                                              │
│  [In-process event bus] ──── persisted, same txn as state ────┐        │
│         │ events                                              │        │
│         ▼                                                     │        │
│  [Orchestrator: fixed DAG, models in the nodes]               │        │
│    ├─ story miner        (deterministic — no model calls)     │        │
│    ├─ angle · copywriter · reviewer   (LLM nodes)             │        │
│    ├─ PrePublishGate     (deterministic — input & output)     │        │
│    └─ publisher + outbox                                      │        │
│         │                        │                            │        │
│         │ KB query               │ dispatch                   │        │
│         ▼                        │                            ▼        │
│  [Knowledge base: FTS5 + embeddings] ───────────────► [SQLite file]    │
│                                  │                    state · content  │
│  [Metrics collector (daily)] ────┤                    outbox · ledger  │
│                                  │                    metrics · events │
└──────────────────────────────────┼─────────────────────────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
      [LLM providers]      [Channel adapters]    [GitHub API]
   Anthropic · OpenAI      auto · assisted        traffic · events
   Ollama (local HTTP)     draft → human
```

## Containers

1. **CLI & review surface** — the command set ([`commands/README.md`](../commands/README.md)) and
   the human approval queue. The review surface is pluggable (CLI, local dashboard,
   Discord/Telegram) because the gate must reach the maintainer where they already are.
2. **In-process event bus** — CloudEvents ([`EVENTS.md`](EVENTS.md)), persisted to the same file in
   the same transaction as the state change. Redis remains an optional adapter for a future hosted
   mode, never a default dependency.
3. **Orchestrator** — a **fixed DAG with models inside the nodes**, not an agent that plans the
   pipeline ([`STATE_MACHINE.md`](STATE_MACHINE.md)). Two nodes are deterministic and hold no model
   at all: the **story miner** and the **PrePublishGate**. That is deliberate — the miner is what
   decouples cost from repository activity
   ([ADR-0013](../adr/ADR-0013-cost-control-and-model-routing.md)), and the gate must not be asked
   to catch what a model produced
   ([ADR-0014](../adr/ADR-0014-deterministic-pre-publish-gate.md)).
4. **SQLite store** — state, content, approvals, outbox, transition ledger, metrics time series,
   event log and knowledge base, in one file ([`DATA_MODEL.md`](DATA_MODEL.md)). Single-writer, WAL
   mode, guarded by the manifest's optimistic-concurrency counter.
5. **Knowledge base** — FTS5 plus an embeddings table in the same file. At a few thousand chunks an
   exhaustive cosine scan is microseconds; a dedicated vector service here answers a question this
   scale never asks.
6. **Publisher & channel adapters** — modular per destination, branching on the profile's tier
   ([ADR-0011](../adr/ADR-0011-channel-capability-tiers.md)). `draft`-tier adapters have **no
   dispatch path at all**; they emit a payload and a composer link.
7. **Metrics collector** — the daily append-only snapshot. The one job whose failure destroys data
   rather than delaying it, because the GitHub Traffic API's window is roughly 14 days
   ([ADR-0015](../adr/ADR-0015-attribution-methodology.md)).

## What is deliberately absent

| Not present | Why |
|---|---|
| Postgres / Redis | No scale here justifies the install cost (ADR-0016) |
| ChromaDB / a vector service | No embedded build exists for this runtime, and the corpus is hundreds of documents |
| LiteLLM / a Python sidecar | Provider integrations are native HTTP; it removes a whole ecosystem from the install |
| A scheduler daemon | Cadence runs from cron or CI. A daemon holding publishing credentials is a standing risk for no gain |
| Any autonomous publish path | Every route to a live post passes through a human `approve` |

The V1 web dashboard and a hosted multi-repository workspace are genuinely different deployments
with a genuinely different store. Saying so now is cheaper than pretending one architecture serves
both.
