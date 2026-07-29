# ADR-0013: Cost control — deterministic pre-filter, capability-tier routing, hard ceilings

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-07-29 |
| **Related** | [ADR-0003 — Multi-agent orchestration](ADR-0003-agent-orchestration.md) (this ADR manages the cost ADR-0003 names but does not bound) · [ADR-0005 — Provider abstraction](ADR-0005-provider-abstraction.md) · [`commands/mine.md`](../commands/mine.md) |

## Context

ADR-0003 puts six agents in a cascade and records the consequence honestly — *"increased LLM token
usage and pipeline latency"* — then stops. Naming a cost is not managing one. As specified, the
pipeline's spend scales with **repository activity**, which is uncorrelated with the number of
stories worth telling: a 200-commit refactoring day costs two orders of magnitude more than a
5-commit week and produces no more content.

Two structural cost bombs sit inside that cascade:

- **Every mined event reaches a model.** The most expensive stage (drafting) runs on candidates the
  system was always going to discard.
- **The reviewer↔copywriter critique loop is unbounded.** Two agents disagreeing politely is the
  classic runaway of this architecture, and it fails *expensively* rather than loudly.

The audience makes this decisive rather than merely untidy. EADROS targets solo open-source
maintainers, who abandon a tool that produces a surprising bill exactly once. And ADR-0005 promises
a provider abstraction spanning hosted APIs and local Ollama, so any routing scheme that hardcodes
one vendor's model names both breaks that promise and rots — the specification already carries a
stale model name from an earlier draft, which is the failure mode in miniature.

## Options considered

**A. Deterministic pre-filter + capability-tier routing + prompt caching + hard ceilings** *(chosen)*

- ✅ **The cheapest token is the one not spent.** [`/eadros mine`](../commands/mine.md) ranks
  candidates with **no model calls at all**; only the top `budget.max_campaigns_per_run` reach a
  model. Cost decouples from repository activity and attaches to *content actually pursued* —
  which is the quantity the maintainer wanted to pay for.
- ✅ **Capability tiers, not model names.** The manifest routes stages to `cheap` / `mid` /
  `strong`; the provider profile maps those to concrete models. The spec stops naming versions, so
  it stops rotting, and ADR-0005's multi-provider promise survives a vendor's renaming.
- ✅ **Spend lands where it buys the most.** `review` gets the strong tier because it is the last
  mechanical check before a human, and human attention is the scarcest resource in the system —
  a miss there costs a person's time and possibly a retraction. `triage` gets the cheap tier
  because it is the highest-volume, lowest-difficulty stage.
- ✅ **Prompt caching on the knowledge base.** Vision, ADRs, positioning, the voice profile and the
  past-post index are stable across runs and dominate the context. Caching them is the single
  largest saving available and costs one design decision.
- ✅ **Ceilings are gates, not advice** — the same discipline as an EADOS hard NFR budget. A
  ceiling with no number fails `/eadros audit`.
- ❌ **The pre-filter can discard a good story a model would have recognised.** Mitigated by making
  the ranking free, inspectable and explainable (`--explain`), and by a golden-set eval measuring
  the miner's recall. A cheap ranker that is *audited* beats an expensive one that is trusted.
- ❌ **Tier mapping is one more thing to maintain per provider.** It lives in the provider profile
  that ADR-0005 requires anyway.

**B. One strong model for every stage** — ✅ trivial, best quality per call; ❌ pays the top rate
for classification work, and the bill still scales with commits rather than with value. **Rejected.**

**C. One cheap model for every stage** — ✅ cheapest, and makes a fully local Ollama run trivial;
❌ guts the reviewer, which is the gate that lets the human gate stay meaningful. A weak reviewer
means every draft's defects reach the maintainer, who then rubber-stamps them — the exact decay
ADR-0014 exists to prevent. **Rejected as the default; permitted as an explicit local-only posture
that the manifest records and `/eadros audit` reports.**

**D. No in-product control; tell the user to set a spend cap at their provider** — ✅ zero work;
❌ the cap fires mid-campaign, leaving content half-published across channels, and it offers no
per-stage attribution, so a maintainer who wants to spend less has no idea what to cut.
**Rejected.**

## Decision

1. **Deterministic pre-filter.** `mine` scores with no model calls; only the top
   `budget.max_campaigns_per_run` (default 5) enter the pipeline.
2. **Capability-tier routing.** `budget.model_routing: {triage: cheap, drafting: mid, review:
   strong}`. The manifest never names a model; the provider profile maps tiers to models.
3. **Prompt caching** on the knowledge-base context, on by default (`budget.prompt_cache: true`).
4. **`budget.max_reviewer_iterations`, default 2, hard.** On exhaustion the draft goes to the human
   with the reviewer's open objections attached, rather than looping.
5. **`budget.cost_per_week` as a hard ceiling**, with `on_exceed: pause` as the default. `pause`
   blocks dispatch; queued work survives.
6. **Attribution per campaign.** `model`, `prompt_version`, input/output tokens and cost are
   recorded per stage. You cannot manage what you do not attribute, and you cannot debug a
   regression without knowing which prompt version produced it.
7. **`--explain` before spending, `--budget-check` to refuse.** The estimate is shown before the
   run, not after.

## Consequences

- Cost tracks stories pursued rather than repository activity, which is the only version of the
  bill a maintainer can reason about or reduce.
- The specification stops naming model versions, so it survives vendor renaming — and the stale
  model reference in the current draft becomes a fixable defect rather than a recurring one.
- A fully local run is viable and its quality cost is **stated** (option C as a recorded posture)
  rather than discovered.
- `mine` must be genuinely good, since it is now the only thing standing between the repository and
  the bill. Its golden-set eval is load-bearing, not optional.
- The store gains per-stage cost columns, so `/eadros digest` can answer *"what did this week cost
  and which stage spent it"* — the question that decides whether the tool stays installed.
