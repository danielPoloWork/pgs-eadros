# Eval suite: `miner` — story ranking

Verifies [`/eadros mine`](../commands/mine.md): does the deterministic scorer surface the commits
worth telling a story about, and does it drop the ones that are not?

**Why this suite is load-bearing.** [ADR-0013](../adr/ADR-0013-cost-control-and-model-routing.md)
made the miner the only thing standing between the repository and the bill — everything it ranks
into the top K gets paid for, everything below the cut is free and invisible. A ranker nobody
measures is a cost decision nobody audits.

The suite is **deterministic**, so a single run per case is sufficient and results are exactly
reproducible. This is the contrast worth noticing against [`reviewer`](reviewer.md): moving the
ranking into deterministic code did not just make it cheap, it made it *provable*.

## Corpus

Real commits from real repositories, labelled by hand.

| Field | Values |
|---|---|
| `verdict` | `story` / `no-story` |
| `archetype` | the shape it should be told as, when `story` |
| `strength` | `strong` / `marginal` — a marginal story missed is not the same failure as a strong one missed |
| `provenance` | `observed` (required — synthesised commits do not carry realistic distribution) |
| `labellers` | ≥ 2 for any case entering the gated set |

**Labelling protocol.** Two maintainers label independently; cases where they disagree are recorded
with `agreement: false` and **excluded from the gated metrics** while remaining in the corpus as
diagnostics. A commit that two experienced people cannot agree is a story is not ground truth — it
is noise, and scoring a ranker against noise produces a number that moves for no reason.

Target composition: ~400 commits, of which ~15% `story`. **The class imbalance is the realistic
one** and must not be corrected away — a suite balanced 50/50 measures a problem the miner never
faces, and will happily pass a scorer that floods the maintainer with candidates.

## Metrics

| Metric | Gate | Why |
|---|---|---|
| **Precision@K** (K = `max_campaigns_per_run`) | ≥ 0.70 | Of what we pay to draft, how much deserved it |
| **Recall on `strong`** | ≥ 0.90 | Missing an obvious story is the failure a maintainer notices |
| **Recall on `marginal`** | ≥ 0.50, advisory | Genuinely ambiguous; a hard gate here would be false rigour |
| **Exclusion safety** | **1.0** | No `story` case may be caught by a hard exclusion |
| **Archetype accuracy** | ≥ 0.75 | Wrong shape wastes a draft even when the story was right |
| **Dedup accuracy** | ≥ 0.95 | Re-proposing last week's story is the failure that gets the tool uninstalled |

**Exclusion safety is the one absolute here.** The hard exclusions — dependency bumps, lockfiles,
formatting, merge commits, vendored paths — are unconditional: a commit they match never reaches a
model and never appears in the cut list, so a false exclusion is **invisible**. Every other mistake
this scorer makes is recoverable by looking at the output. This one is not, which is why it is gated
at 1.0 while precision is gated at 0.70.

## Baseline

**Rank by diff size.** The miner must beat it on Precision@K by a clear margin. If a weighted signal
model cannot outperform "biggest change wins", the weights are decoration and the honest move is to
delete them rather than tune them.

A second baseline worth tracking: **rank by recency**. It is what a maintainer does unaided, and
beating it is the minimum claim the product makes.

## What the suite runs

1. **Ranking.** Score the corpus, compare the top K against labels, report all metrics per class.
2. **Exclusion regression.** Every hard-exclusion category has cases asserting a score of exactly
   zero. A dependency bump that scores above zero is a **test failure, not a judgment call**.
3. **Consent propagation.** With `postmortem: false` in the voice profile, assert no revert-driven
   candidate is offered as a post-mortem, and that candidates whose every archetype is disabled are
   dropped with a recorded reason
   ([ADR-0012](../adr/ADR-0012-voice-profile-and-calibration.md)).
4. **Dedup.** Replay a story already in `kb_documents` as a past post; assert `duplicate`. Replay it
   with the underlying work advanced; assert `supersedes` rather than `duplicate` — a legitimate
   follow-up must not be suppressed as a repeat.
5. **Gate interaction.** Candidates failing the input-pass gate (embargo, path allowlist) never
   appear in output, at any score.
6. **Determinism.** The same corpus scored twice produces byte-identical output. A miner that drifts
   is a miner that has acquired a model call somewhere it should not have.

## Weight tuning is an experiment, not a preference

Scoring weights live in `config/scoring.yaml` precisely so that changing them is a measurable act.
A weight change is proposed as a diff, evaluated against the corpus, and accepted or rejected on the
metrics table — not on whether the new ranking looks nicer on this week's commits.

`/eadros eval --suite miner --baseline` prints the current weights, the proposed weights, both
metric tables and the deltas per class. **Per class**, because a weight change that lifts aggregate
precision while dropping `strong` recall is a regression wearing a win.
