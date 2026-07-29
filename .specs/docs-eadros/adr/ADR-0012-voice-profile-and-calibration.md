# ADR-0012: Voice is elicited by sample and enforced by lint, not described by adjectives

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-07-29 |
| **Related** | [`orchestrator/voices/_schema.md`](../orchestrator/voices/_schema.md) · [`orchestrator/interview.md`](../orchestrator/interview.md) (Phase 3) · [ADR-0008 — Human review gate](ADR-0008-human-review.md) |

## Context

EADROS's thesis is that automated content can be authentic because it is derived from real
engineering work. That thesis has one failure mode, and it is fatal rather than gradual: **if
readers recognise the output as model-generated, the authenticity claim is falsified in a single
post**, and no subsequent post recovers it. Developer audiences are unusually good at this
recognition and unusually unforgiving of it.

The conventional way to control tone is a configuration field — `tone: professional-but-friendly`.
This does not work, for a specific reason: every model already believes it writes that way. The
adjective selects nothing, so the output lands in the uniform register that *is* the tell. Worse,
the failure is invisible from inside: the config looks configured, the drafts look fine to their
author, and the audience sees generated prose.

There is a second, related gap. [ADR-0008](ADR-0008-human-review.md) claims the human review gate
eliminates hallucination risk. It does not — reviewers rubber-stamp by the third week, and a gate
whose only enforcement is human attention decays predictably. Whatever controls voice must be
mechanical enough to run before a human sees the draft, so that human attention is spent on
judgment rather than on catching the same buzzword for the twentieth time.

## Options considered

**A. Sample-derived fingerprint + deterministic construction lint + a calibration loop** *(chosen)*

Ask the maintainer for two or three things they have actually written. Extract measurable
properties into `voices/<slug>.yaml`. Enforce a forbidden-**construction** list mechanically. Close
the interview by generating three drafts from a real commit, having the maintainer edit one, and
folding the diff back into the profile.

- ✅ **Every field is falsifiable.** Mean sentence length, sentence-length variance, code-to-prose
  ratio, hedging frequency, opening move — each is a number or an enum a check can evaluate.
  "Approachable" cannot be checked; these can, which is what makes the profile a specification
  rather than a wish.
- ✅ **The construction lint catches what word lists miss.** Banning *revolutionary* is easy and
  nearly worthless. The actual tells are structural — the rhetorical-question opener, the
  "it's not X, it's Y" antithesis, the parallel triad where the content supports two items, the
  closing paragraph that restates the post. These are enumerable, and a deterministic check on them
  **survives a model change**, which no prompt instruction does.
- ✅ **The calibration loop converts a self-description into a measurement.** What the maintainer
  *cuts* from a draft is far more informative than what they said they wanted, and it is exactly
  what populates `forbidden.constructions`. Nobody can introspect their own writing well enough to
  list these; everybody can recognise them on sight and delete them.
- ✅ **It composes with the approval gate.** `approval_mode: edit-required` means the maintainer is
  already editing drafts, so the calibration signal keeps arriving for free after onboarding. The
  governance control and the quality control feed each other rather than competing for attention.
- ❌ **Requires the maintainer to produce samples**, which some cannot at the start. Mitigated by
  seeding from a shipped profile — and requiring that the seed be **named explicitly**, so a
  borrowed voice is never presented as their own.
- ❌ **A fingerprint can overfit to three samples.** Mitigated by tolerances rather than exact
  targets, and by re-runnable recalibration (`/eadros onboard --recalibrate`) whenever the
  maintainer notices they are rewriting heavily.

**B. Adjective/enum tone configuration** — ✅ trivial to build, familiar, one line of config;
❌ selects nothing the model was not already doing, cannot be verified, cannot be reviewed, and
produces exactly the register the product exists to avoid. It is the option that *looks* like a
feature while solving none of the problem. **Rejected.**

**C. Fine-tune a model on the maintainer's corpus** — ✅ genuinely captures voice, the strongest
result available; ❌ needs far more text than a maintainer has written, re-tuning on every drift,
per-provider lock-in that breaks [ADR-0005](ADR-0005-provider-abstraction.md)'s multi-provider
requirement (including local Ollama), and costs orders of magnitude more than the entire pipeline.
Also unfalsifiable in the same way as B: a tuned model that drifts gives no signal that it has.
**Rejected as disproportionate; revisit if per-maintainer adaptation becomes cheap.**

**D. No voice configuration; let the Reviewer Agent judge prose quality** — ✅ nothing to elicit,
nothing to maintain; ❌ asks one model to detect the register that models share, against no stated
standard. The reviewer would have to *infer* what the maintainer sounds like from the same corpus
it is reviewing. Unbounded, unmeasurable, and untestable — there is no golden set for "sounds
right" without a profile to score against. **Rejected.**

## Decision

Voice is captured in `voices/<slug>.yaml`, derived from the maintainer's own writing:

1. **Elicit by sample.** Phase 3 asks for two or three real artifacts and extracts a measurable
   fingerprint. Every derived field carries provenance **`inferred_from_sample`** — a value that is
   neither asked nor defaulted — and **must be shown back in plain language before adoption**. An
   inferred value the maintainer never saw is indistinguishable from a guess.
2. **Enforce by lint.** `forbidden.words` and, more importantly, `forbidden.constructions` run
   deterministically before any human sees a draft. A hit is a rewrite, not a warning.
3. **Consent per archetype.** `postmortem` and `opinion` default **off** and must be asked.
   Publishing a failure story or a stated opinion under someone's name is a reputational act, not a
   content type.
4. **Calibrate against reality.** The phase ends by generating three drafts from a real commit,
   taking the maintainer's edit, and diffing it into the fingerprint. Two rounds is normal;
   converging on the first is flagged as suspicious.
5. **Claims stay strict independently of voice.** Every factual claim carries a resolvable
   `source_ref`, and code snippets are extracted verbatim at a SHA, never generated. Voice governs
   how it reads; claim discipline governs whether it is true.

## Consequences

- The authenticity risk is managed by a mechanism instead of an intention, and the mechanism
  produces evidence: `voice.calibration[]` records that the profile was corrected against real
  maintainer edits rather than asserted.
- The Reviewer Agent gets a **stated standard to review against**, which makes a golden-set eval
  possible — precision and recall on planted hype and planted constructions. Without ADR-0012 there
  is nothing to score.
- ADR-0008's overclaim is repaired without weakening the gate: the deterministic lint runs first,
  so human review spends its attention on judgment rather than on catching known-bad patterns.
- Onboarding is longer, and deliberately so — Phase 3 ends on a sample, not an answer. A program
  still running on an uncalibrated seed profile after two weeks is one whose voice was never
  actually elicited, and `/eadros audit` reports it.
- The forbidden-construction list is the profile's most valuable artifact and grows over the life
  of the program, since `edit-required` approvals keep supplying deletions to learn from.
