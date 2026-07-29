# EADROS Verification & Eval Strategy

The specification makes four load-bearing claims: *a human approved this*, *we never publish where
we are not allowed to*, *we never leak*, and *we never double-post*. This directory is how each one
is checked rather than asserted.

## The organising decision: safety lives in deterministic code

The system has components of four kinds, and they need genuinely different verification. Conflating
them is why most agentic systems end up with an eval suite that measures the easy thing.

| Class | Components | Method | Deterministic |
|---|---|---|---|
| **Safety-critical** | PrePublishGate, state machine, outbox, tier routing | Unit + property tests | ✅ |
| **Ranking** | Story miner scoring | Golden set, labelled corpus | ✅ |
| **Generative** | Angle, Copywriter, Reviewer | Golden set, N runs, variance-aware | ❌ |
| **Integration** | Channel adapters, platform APIs | Snapshot + recorded contract | ✅ |

**Every safety guarantee sits in a deterministic component on purpose.** That is the payoff of
[ADR-0014](../adr/ADR-0014-deterministic-pre-publish-gate.md): because the gate is code rather than
a prompt, its recall on planted secrets is a **test result**, not a hope. Had the leak control lived
in the Reviewer Agent's instructions, verifying it would mean proving a property of a language
model — which is not a thing you can do, at any budget.

The generative components are verified too, but what they are trusted with is scoped accordingly:
they affect *quality*, never *permission*.

## Suites

| Suite | Verifies | Blocks CI |
|---|---|---|
| [`miner`](miner.md) | Story ranking precision and recall, exclusion safety | ✅ |
| [`reviewer`](reviewer.md) | Defect detection per class, false-positive rate | ✅ (lower bound) |
| [`adversarial`](adversarial.md) | Prompt injection containment, leak recall | ✅ **hard** |
| [`channels`](channels.md) | Formatter conformance, platform contract handling | ✅ |
| [`cost`](cost.md) | Budget adherence, iteration caps, cache hit rate | ✅ |
| `state` | The six property tests in [`STATE_MACHINE.md`](../architecture/STATE_MACHINE.md) | ✅ |

## Discipline

Four rules apply to every suite. They are what separate an eval that measures progress from one
that produces a number.

**1. Variance, not point estimates.** Generative suites run **N ≥ 20** times per case. Report mean
and standard deviation, and **gate on the lower confidence bound, never the mean.** A prompt change
that improves the average while doubling the variance has made the system worse, and a single
passing run is not evidence of anything.

**2. Every component beats a trivial baseline, or it is not earning its cost.** Each suite declares
one:

| Component | Baseline it must beat |
|---|---|
| Story miner | Rank by diff size |
| Reviewer | Regex over the forbidden-word list |
| Angle selection | Always pick the most frequent archetype |
| Copywriter | The channel's release-note template, filled |

A model-based stage that does not clearly beat its baseline should be deleted, not tuned. This is
the check that keeps [ADR-0003](../adr/ADR-0003-agent-orchestration.md)'s six agents honest — a
multi-agent architecture is a cost decision, and each agent has to justify its share.

**3. Per-class metrics, never aggregate.** An aggregate score hides the only thing worth knowing.
A reviewer that catches 100% of buzzwords and 20% of subtly-false technical claims scores well in
aggregate and is useless at the job that matters. Every suite reports **per defect class**, and the
gate is per class.

**4. The corpus is a versioned artifact with provenance.** Each case records where it came from
(`synthesised` / `observed` / `from_retraction`), who labelled it, when, and — for multi-labeller
classes — the **inter-rater agreement**. If two maintainers disagree on whether a commit is a story,
that label is noise and must be known to be noise before a model is scored against it. A corpus that
grows by accretion without provenance quietly stops meaning anything.

## Where cases come from

The corpus grows from three sources, and the third is the important one:

- **Synthesised** — planted defects with known ground truth. Fast to produce, easy to overfit to.
- **Observed** — real repository history and real drafts, labelled by hand. Slow, and the only
  source of realistic distribution.
- **From retractions** — every `/eadros retract` writes its case back
  ([`retract.md`](../commands/retract.md)). A `leak` that reached publication is a **gate defect**,
  and its regression test asserts that this exact content would now be blocked.

The third source is the only mechanism by which the system gets safer from having been wrong, and
it is why `retractions.gate_verdict_id` exists in the schema.

## CI policy

- **Deterministic suites block on any regression.** They are cheap and exact; there is no reason to
  tolerate a failure.
- **Generative suites block on the lower bound crossing the per-class threshold**, evaluated against
  the prompt version being merged. Results are recorded per `prompt_version`, so a regression is
  attributable to a change rather than to the weather.
- **The adversarial suite is a hard gate with no threshold negotiation** on its secrets class: recall
  must be **1.0**. Everywhere else a number below 100% is a trade-off; here it is a defect.
- Model calls in CI use recorded fixtures by default. A nightly job runs the generative suites
  against live providers, because fixtures drift from model behaviour and a suite that only ever
  sees fixtures eventually measures the fixtures.

## What is deliberately not tested

Stating this matters as much as the suites, because an eval directory implies coverage it does not
have:

- **Whether a post lands well with developers.** This is the outcome
  [ADR-0015](../adr/ADR-0015-attribution-methodology.md) shows is only directionally measurable, at
  a cadence that never reaches statistical power. No suite here claims it, and none should.
- **Whether the voice is authentically the maintainer's.** The fingerprint's *mechanical* properties
  are checked ([`reviewer`](reviewer.md)); whether the result sounds like them is settled by the
  calibration loop with a human, not by a metric
  ([ADR-0012](../adr/ADR-0012-voice-profile-and-calibration.md)).
- **Whether a platform's terms still permit what a profile claims.** That is a human re-verification
  on a 90-day clock, surfaced by `/eadros doctor`. A test cannot read a terms-of-service page and
  understand it.

Each of these is a place where the honest answer is a human process, and dressing it as a metric
would make the whole directory less trustworthy.
