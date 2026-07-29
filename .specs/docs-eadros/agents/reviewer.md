# Agent: Reviewer

| | |
|---|---|
| **Tier** | `strong` |
| **Position** | Node 5 — critiques the Copywriter, before the gate's output pass and the human queue |
| **Decides** | Whether a draft is fit to reach a human |

The strong tier goes here, and the reasoning is worth stating because it looks backwards: the
Reviewer is **the last mechanical check before a person**, and human attention is the scarcest
resource in the system. A miss here costs a maintainer's time, and occasionally a retraction.

## Input

Draft, `claims[]`, voice profile, channel profile, positioning, and the **deterministic resolver's
verdicts** on every registered claim.

## Output

```json
{
  "verdicts": [
    {"class": "construction", "span": [0, 62],
     "finding": "rhetorical-question opener", "severity": "block"},
    {"class": "unregistered_assertion", "span": [410, 470],
     "finding": "'3x faster' asserted with no claim registered", "severity": "block"}
  ],
  "recommendation": "revise" | "pass",
  "open_objections": []
}
```

## The two things only a model can do here

Most of what the Reviewer is credited with in earlier drafts is not its job, and saying so sharpens
what is:

**1. Unregistered assertions — the resolver's blind spot.** The claim resolver is deterministic and
exact: it checks that each declared `source_ref` exists at the stated SHA and matches. But **it can
only check claims that were declared.** A sentence asserting *"this made startup 3× faster"* with no
entry in `claims[]` is invisible to the resolver — mechanically clean, factually unbacked.

Finding factual assertions the Copywriter made without registering them is the genuinely model-shaped
task in this pipeline, and it is the one the whole claim discipline rests on.

**2. Plausible-but-wrong technical claims.** A registered claim whose `source_ref` resolves can still
mischaracterise what the source says — *"the escrow counter uses vector clocks"*, citing a commit that
implements HLCs. This is the hardest thing asked of any component here, and
[its eval gate is set at 0.60](../eval/reviewer.md), honestly, where the component actually performs.

That low number is itself the argument for `claim_discipline: strict`: **the system's real defence
against false claims is requiring a resolvable source for every one of them, not hoping a model
notices.** The Reviewer is the second line, not the first.

## Constraints

**It never edits.** It emits verdicts with spans; the Copywriter rewrites. This separation is not
tidiness — if the Reviewer silently fixed what it found, detection would be unmeasurable (you cannot
score a catch that left no trace) and the human would lose sight of what was wrong with the draft
they are approving.

**It critiques against a stated standard, never taste.** The standard is the voice profile, the claim
discipline, the positioning, and archetype consent. This is what
[ADR-0012](../adr/ADR-0012-voice-profile-and-calibration.md) made possible: before a fingerprint
existed there was no standard, so "the reviewer works" was unfalsifiable.

**It does not enforce safety.** Secrets, deny-terms, paths, embargo and taint are the
[PrePublishGate](../adr/ADR-0014-deterministic-pre-publish-gate.md)'s, in deterministic code, gated
at recall 1.0. **Asking a model to catch what a model produced is not a control** — and a Reviewer
credited with safety would let the real gate look optional.

**It is bounded.** At most `budget.max_reviewer_iterations` rounds (default 2). On exhaustion the
draft advances to the human with `open_objections` populated — never looping, never silently
passing. An unbounded critique loop is this architecture's classic runaway, and it fails expensively
rather than loudly.

## Defect classes

Reported and gated **per class**; the aggregate is never computed, because it is the number that
hides the problem — a Reviewer catching 100% of buzzwords and 20% of false claims scores well and is
useless at the job that matters.

| Class | Gate (recall, LCB) | Who really owns it |
|---|---|---|
| `hype_word` | ≥ 0.98 | Deterministic lint; the Reviewer is a second pass |
| `construction` | ≥ 0.85 | **Reviewer** |
| `unregistered_assertion` | ≥ 0.85 | **Reviewer** — the resolver cannot see these |
| `unresolvable_claim` | 1.0 | Deterministic resolver; failure means the resolver is broken |
| `false_technical` | ≥ 0.60 | **Reviewer**, honestly hard |
| `fingerprint_drift` | ≥ 0.80 | **Reviewer** |
| `archetype_violation` | 1.0 | Deterministic — consent is checked against the manifest |
| `clean` (control) | FP ≤ 0.10 | — |

**The false-positive gate is as tight as most recalls, deliberately.** A Reviewer that rejects good
work teaches the maintainer to override it, and an overridden gate is an absent gate — a failure mode
invisible to any suite that only measures detection.

## Verification

[eval/reviewer.md](../eval/reviewer.md). N ≥ 20 runs per case, gated on the lower confidence bound.

**Baseline:** a regex over `forbidden.words`. It will score near-perfectly on `hype_word` and zero
everywhere else, which is the point — this agent's justification is `construction`,
`unregistered_assertion`, `false_technical` and `fingerprint_drift`. If it is not clearly ahead on
those, it is an expensive regex.

**Corpus growth is free.** Every human rejection is a labelled false negative — a draft a person
threw out that the Reviewer passed — written back by `/eadros reject` with its reason. It is the most
valuable case type available, because nobody had to invent it.
