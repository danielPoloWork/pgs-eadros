# Eval suite: `reviewer` — defect detection

Verifies the Reviewer Agent: does it catch what it is supposed to catch, and does it leave clean
drafts alone?

This is the suite that [ADR-0012](../adr/ADR-0012-voice-profile-and-calibration.md) makes possible.
Before a voice profile existed there was no stated standard, so "the reviewer works" was
unfalsifiable. With a fingerprint and a forbidden-construction list, the reviewer has something to
be *wrong about*.

**Generative, therefore variance-aware.** N ≥ 20 runs per case, gated on the lower confidence bound.

## Defect classes

The corpus plants exactly one known defect per case. **Metrics are reported per class and gated per
class** — the aggregate is not reported at all, because it is the number that hides the problem.

| Class | Example | Gate (recall, LCB) |
|---|---|---|
| `hype_word` | *"a revolutionary approach"* | ≥ 0.98 |
| `construction` | rhetorical opener; *"it's not X — it's Y"*; the parallel triad | ≥ 0.85 |
| `unregistered_assertion` | *"3× faster"* asserted with no entry in `claims[]` | ≥ 0.85 |
| `unresolvable_claim` | a SHA that does not exist; a benchmark number that does not match the run | **1.0** |
| `false_technical` | plausible but wrong — *"the escrow counter uses vector clocks"* | ≥ 0.60 |
| `fingerprint_drift` | mean sentence length outside tolerance; uniform variance; emoji headers | ≥ 0.80 |
| `archetype_violation` | an opinion asserted where `opinion: false` | **1.0** |
| `clean` (control) | no defect | FP rate ≤ 0.10 |

Four of these gates deserve their justification stated:

**`unregistered_assertion` is the class the resolver structurally cannot cover**, and therefore the
one that most justifies a model at this position. The resolver checks declared claims exactly; a
factual sentence the Copywriter never registered is invisible to it — mechanically clean, factually
unbacked. Finding those is the genuinely model-shaped task in the pipeline
([agents/reviewer](../agents/reviewer.md)).

**`unresolvable_claim` is gated at 1.0 and is not really a model task.** The check is mechanical —
does the SHA resolve, does `file:line` exist at that SHA, does the number match the benchmark
record. It is scored here because the reviewer is where it is *invoked*, but a failure means the
resolver is broken, not that the model was unlucky. A model-shaped gate on a deterministic check
would be a category error.

**`false_technical` is gated at only 0.60, honestly.** Detecting a plausible-but-wrong technical
claim is the hardest thing asked of this system, and a gate set where the component actually
performs is more useful than an aspirational one that gets waived every sprint. The low number is
also the argument for `claim_discipline: strict`: the system's real defence against false claims is
requiring a resolvable `source_ref` for every one of them, not asking a model to notice.

**The false-positive rate on clean drafts is gated as tightly as most recalls.** A reviewer that
rejects good work teaches the maintainer to override it, and an overridden gate is an absent gate.
This failure mode is invisible to any suite that only measures detection.

## Baseline

**A regex over `forbidden.words`.** It will score near-perfectly on `hype_word` and zero everywhere
else — which is exactly the point. The reviewer's justification is `construction`,
`false_technical` and `fingerprint_drift`; if it is not clearly ahead on those, it is an expensive
regex and should be replaced by the cheap one.

## Corpus construction

~200 cases, ~30% controls. Drafts are **real generated output**, not hand-written prose: a corpus of
human-authored drafts measures a distribution the reviewer never sees in production.

Defects are planted by mutation — take a clean draft that passed human review, introduce one defect,
record the mutation. This gives ground truth that is exact and a distribution that is realistic.

**Every rejection recorded in production feeds the corpus.** `rejections.reason` is written by
`/eadros reject` for exactly this: a draft a human rejected but the reviewer passed is a labelled
false negative, and it is the most valuable case type available because nobody had to invent it.

## What the suite runs

1. **Detection.** N runs per case; per-class precision, recall, mean ± σ, lower bound.
2. **Control set.** False-positive rate on clean drafts.
3. **Stability.** Variance per case. **A case whose verdict flips across runs is reported
   separately** — high-variance cases are where prompt changes will produce phantom improvements,
   and they must not be quietly averaged into a passing score.
4. **Iteration bound.** Assert the reviewer↔copywriter loop terminates within
   `budget.max_reviewer_iterations`, and that on exhaustion the draft advances to `pending_approval`
   **carrying its open objections** rather than looping or silently passing
   ([ADR-0013](../adr/ADR-0013-cost-control-and-model-routing.md)).
5. **Profile sensitivity.** Run the same drafts under two different voice profiles; assert
   `fingerprint_drift` verdicts change accordingly. A reviewer that returns the same answer whatever
   the profile says is ignoring the profile — a failure no per-class recall would reveal.
6. **Per-`prompt_version` tracking.** Every result is recorded against the prompt version, so a
   regression is attributable to a change rather than to model drift.

## Reading a result

A passing aggregate with a regressed `false_technical` class is a **regression**, and the report is
laid out to make that obvious rather than discoverable. The gate evaluates each class independently;
there is no averaging across classes and no compensating a drop in one with a gain in another.
