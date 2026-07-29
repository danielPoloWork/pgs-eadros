# Design Patterns Catalogue

Living index of every design pattern **adopted**, **planned**, **considered and rejected**,
or **under evaluation** for `pgs-eadros`. Mandatory reading whenever a PR introduces
or removes a pattern, and updated in the same PR.

- **Rules** — [`AGENTS.md`](../../../AGENTS.md) §8.
- **Canonical taxonomy** — [`design-patterns.md`](design-patterns.md). All pattern names
  used here, in ADRs, and in commit messages must match its spelling and categorisation.

## Architecture style

**Committed style:** Event-Driven — from [`design-patterns.md`](design-patterns.md) §5.
**Pattern discipline:** `advisory` — `advisory` means the agent advises and the human
decides; `enforced` makes conformance to the committed style + adopted patterns a review expectation.


## How to use this catalogue

- **Adding a pattern** — when a PR lands one, add a row to *Implemented / Planned* as
  `Implemented`, with the ADR link and the code location (a real path under
  `.eadros-core/src/main/typescript/...`); a pattern decided in an ADR but not yet in code is added as `Planned`.
- **Refining** — update the row and link the new ADR.
- **Rejecting** — add it to *Rejected* with the reason; do not silently drop it.
- **Removing** — move the row to *Superseded*, link the superseding ADR, keep the history.

Status vocabulary: `Planned` (decided in an ADR, not yet landed) · `Implemented` (present
in `.eadros-core/src/main/...`, ADR `Accepted`) · `Considered` · `Rejected` · `Superseded`.

## Implemented / Planned

_Patterns named in the spec at intake are seeded below as **Planned**; each becomes
**Implemented** with its ADR and a real code location in the PR that introduces it._

| # | Pattern | Status | Problem it addresses | Code location | ADR / PR |
|---|---------|--------|----------------------|---------------|----------|
| — | Outbox | Planned | Dispatch commits intent before the external call; UNIQUE(idempotency_key) makes double-publish structurally impossible rather than procedurally avoided. | _TBD_ | _spec (intake)_ |
| — | Adapter | Planned | One module per destination, branching on the channel profile's tier; draft-tier adapters expose no dispatch path at all (ADR-0007, ADR-0011). | _TBD_ | _spec (intake)_ |
| — | Fixed DAG pipeline with models in the nodes | Planned | The orchestrator never lets a model plan the graph — that is what keeps cost bounded and the trail auditable (COMPONENTS.md). | _TBD_ | _spec (intake)_ |
| — | State machine with a transition ledger | Planned | Campaign and post have separate lifecycles, because a partial publish — live on one channel, awaiting a human on another, failed on a third — is the normal outcome (STATE_MACHINE.md). | _TBD_ | _spec (intake)_ |
| — | Persisted event log with correlation and causation ids | Planned | Events are written in the same transaction as the state change, so one query reconstructs how a CI failure became three posts (ADR-0002, EVENTS.md). | _TBD_ | _spec (intake)_ |
| — | Provider abstraction with tier routing | Planned | Hosted and local (Ollama) model providers sit behind one interface, which is where the cost ceiling is enforced (ADR-0005, ADR-0013). | _TBD_ | _spec (intake)_ |


## Rejected

_No rejections recorded yet._

| # | Pattern | Considered for | Rejected because | ADR / PR |
|---|---------|----------------|------------------|----------|
| — | —       | —              | —                | —        |

## Superseded

_No superseded patterns yet._

| # | Pattern | Superseded by | When | ADR / PR |
|---|---------|---------------|------|----------|
| — | —       | —             | —    | —        |

## Candidate patterns to consider

The taxonomy in [`design-patterns.md`](design-patterns.md) lists every pattern in scope. As
the architecture takes shape, narrow that universe to the patterns plausibly applicable to
*this* artifact and list them here by category, each with a one-line "possible application".
A candidate remains a candidate until adopted (own ADR) or explicitly rejected.

## Out-of-scope categories

Record here any taxonomy category pre-classified as not applicable to this artifact (with a
one-line reason), so the policy of explicit rejection is honoured without filling the
*Rejected* table with N/A noise.
