# Agent: Copywriter

| | |
|---|---|
| **Tier** | `mid` |
| **Position** | Node 4 — after Angle, in a bounded loop with the Reviewer |
| **Decides** | How the story reads, per channel and per locale |

One draft per `(channel, locale)`. The channel profile and the voice fingerprint are **constraints on
the output**, not suggestions in a prompt — both are re-checked deterministically afterwards
([channels suite](../eval/channels.md)), so a violation here is caught rather than shipped.

## Input

| Field | Notes |
|---|---|
| `angle` | Archetype, hook, thesis, evidence refs |
| `voice` | Fingerprint, forbidden words **and constructions**, person, byline |
| `channel_profile` | `max_chars`, markdown flavour, `code_blocks`, tags, thread support |
| `locale` | Plus its `mode`: `translated` or `authored` |
| `snippets` | **Extracted verbatim from the repository at a stated SHA** |
| `kb_context` | Recent posts, so this one does not repeat them |

## Output

```json
{
  "body": "the draft",
  "claims": [
    {"span": [140, 198], "text": "the gate blocked a boundary violation",
     "source_kind": "commit", "source_ref": "a1b2c3d"}
  ],
  "snippets_used": ["snip-1"],
  "tags": ["architecture", "testing"],
  "char_count": 2840,
  "constraint_report": []
}
```

## Constraints

**It never writes code.** Snippets arrive pre-extracted, verbatim, at a SHA; the Copywriter may
choose which to include and how to introduce them, and may not alter a character. This single rule
removes the largest hallucination surface in the system and is what makes deterministic claim
resolution possible at all — you cannot verify a snippet against a repository that never contained
it.

**Every factual assertion is emitted as a registered claim.** The `claims[]` array is produced *by
this agent*, not extracted from its prose afterwards. The discipline is deliberate and slightly
uncomfortable: **if it cannot cite it, it cannot write it.** A resolver can only check claims that
were declared, so an assertion made without registering it is invisible to the mechanical check —
which is exactly the gap the [Reviewer](reviewer.md) is pointed at.

**The forbidden list is a hard constraint, and the constructions half matters more than the words.**
Avoiding *revolutionary* is easy and nearly worthless. The tells are structural: the
rhetorical-question opener, the "it's not X — it's Y" antithesis, the scene-setting preamble, the
parallel triad where the content supports two items, the closing paragraph that restates the post
([ADR-0012](../adr/ADR-0012-voice-profile-and-calibration.md)).

**Channel rules are mechanical.** No fenced code where `markdown: none` — LinkedIn renders it as
literal backticks, which is the class of mistake that makes automated posting recognisable at a
glance. Length within `max_chars`. Tags within `limits.tags`. Threads only where supported.

**Locale mode changes the job.** `translated` derives from the canonical draft — correct for release
notes and factual posts. `authored` writes natively for that audience, which is necessary for
anything whose value is voice, because a translated opinion reads as translated.

## Failure modes

**Never silently exceed a constraint.** When the story cannot be told within the channel's limits,
emit the best draft plus a populated `constraint_report` naming what was violated and by how much.
A truncated post that fits is worse than a flagged one that does not — the first ships broken, the
second gets a decision.

| Situation | Behaviour |
|---|---|
| Content does not fit `max_chars` | Draft + `constraint_report`; the Reviewer sees it, the human decides |
| A needed snippet was not extracted | Write around it, or report the gap — **never reconstruct it from memory** |
| The angle's evidence does not survive the channel's format | Report; a claim that cannot be shown should not be asserted |
| Iteration cap reached | Advance with the Reviewer's open objections attached |

## Verification

- **Formatter conformance** — [channels suite](../eval/channels.md): length, markdown flavour, code
  blocks, tags, canonical URL. Snapshot plus a property test that no output ever exceeds
  `max_chars`, for any input.
- **Fingerprint drift** — [reviewer suite](../eval/reviewer.md), class `fingerprint_drift` (≥ 0.80):
  sentence length and variance, formatting habits, opening move.
- **Baseline:** the channel's release-note template, filled. If a mid-tier model does not clearly beat
  a filled template, the template is cheaper and the stage should be deleted rather than tuned.
