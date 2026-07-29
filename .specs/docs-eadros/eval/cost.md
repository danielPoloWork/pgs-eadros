# Eval suite: `cost` — budget adherence and spend regression

Verifies [ADR-0013](../adr/ADR-0013-cost-control-and-model-routing.md) mechanically. Runs a fixed
corpus end-to-end against a **mock provider that counts tokens and returns canned completions**, so
the suite is deterministic, free, and safe to run on every commit.

**Why cost gets its own suite.** A prompt change that doubles the context window improves nothing a
quality metric can see and doubles the bill. It passes `reviewer`, passes `miner`, passes review —
and shows up six weeks later as a maintainer uninstalling the tool. Spend is a regression class with
no other detector.

## Assertions

| Assertion | Gate |
|---|---|
| Cost per campaign ≤ the corpus baseline + 10% | Blocks |
| Reviewer iterations ≤ `budget.max_reviewer_iterations` | **Hard** |
| Only the top `max_campaigns_per_run` candidates reach a model | **Hard** |
| `mine` issues **zero** model calls | **Hard** |
| Prompt-cache hit rate on KB context ≥ 0.80 | Blocks |
| Each stage routes to its configured tier | Blocks |
| `budget.on_exceed: pause` halts before the ceiling is crossed, not after | **Hard** |

Four of these are hard gates because each one, if it fails, removes a *structural* cost bound rather
than degrading a number:

**`mine` issues zero model calls** is the suite's most important assertion. It is the property that
decouples spend from repository activity, and it is easy to lose by accident — someone reaches for a
model to break a scoring tie, the tests still pass, and the cost model silently inverts. Asserted by
counting calls on the mock provider during a full mine of a 500-commit corpus: the expected number
is zero, not "few".

**Iteration cap** — the unbounded critique loop is this architecture's classic runaway, and it fails
expensively rather than loudly.

**Top-K only** — the pre-filter is the whole cost argument. A path that drafts a candidate below the
cut is the cost model quietly not applying.

**Pause before the ceiling** — a ceiling enforced after the fact is a report, not a gate, and it
leaves content half-published across channels.

## Baseline and drift

The corpus records cost per campaign per prompt version. `/eadros eval --suite cost --since <ver>`
prints the delta, broken out **per stage**:

```
stage      calls   in_tok   out_tok   cost      Δ vs v3
triage        42   18,400     1,260   €0.021    ─
drafting      12   96,800    14,300   €0.418    +38%  ← context grew
review        12  104,200     3,100   €0.512    +2%
                                      ─────────
                            campaign  €0.079/ea  +21%
```

Per stage, because an aggregate rise tells you spending increased and nothing about where. The
example above is the realistic shape of the failure: drafting context grew, quality metrics did not
move, and only this table names it.

## Model routing

Assert each stage resolves to its configured tier, and that **the manifest never names a model**.
A concrete model id appearing anywhere outside a provider profile is a test failure — that is the
coupling ADR-0013 removed, and it is how the specification stops carrying stale model names.

Routing is verified through the provider abstraction with three profiles: a hosted multi-tier
provider, a second hosted provider with different tier names, and **a local Ollama profile where all
three tiers map to one model**. The local case must run to completion with its quality cost recorded
as a stated posture — a fully local run is a supported configuration, and a suite that only
exercises multi-tier hosted providers would let it break silently.

## What this suite cannot tell you

Token counts from a mock provider are structurally accurate and **numerically approximate**: real
tokenisers differ per model, and real completions vary in length. The suite catches *regressions*
and *structural violations* reliably; it does not predict the bill.

The bill comes from `budget_ledger`, which records actual spend per stage from production runs.
`/eadros digest` reports it, and the two numbers should be compared periodically — a growing gap
between modelled and actual cost means the corpus has drifted from real usage and needs refreshing.
