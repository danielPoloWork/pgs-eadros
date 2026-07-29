# Agent: Angle

| | |
|---|---|
| **Tier** | `cheap` |
| **Position** | Node 3 — after the miner and the gate's input pass, before the Copywriter |
| **Decides** | Which story to tell about a candidate, **and whether there is one at all** |

Classification with justification over a small input. It is the highest-volume model stage in the
pipeline, which is why it runs on the cheap tier and why its ability to **decline** matters more than
its ability to choose well.

## Input

| Field | Notes |
|---|---|
| `candidate` | Source refs, signals with weights, diff summary — from [`mine`](../commands/mine.md) |
| `audience` | `program.audience` — the reader with the problem |
| `positioning` | Category, differentiator, comparables |
| `archetypes` | **Only the consented set** — see the constraint below |
| `kb_context` | Past posts on adjacent topics, for non-repetition |
| `channels` | Which destinations this program publishes to |

## Output

```json
{
  "verdict": "angle" | "no_angle",
  "archetype": "tradeoff",
  "hook": "one sentence a reader would stop for",
  "thesis": "what the post argues, in one sentence",
  "evidence_refs": [{"kind": "commit", "ref": "a1b2c3d", "supports": "hook"}],
  "channels": ["devto", "linkedin"],
  "reason": "required when verdict is no_angle"
}
```

## Constraints

**Disabled archetypes are absent from the input, not forbidden in the prompt.** If the maintainer
left `postmortem: false`, the option does not appear in `archetypes` at all. This is the difference
between a structural guarantee and an instruction — **prohibitions leak under pressure; absent
options cannot be chosen.** The consent decision from
[ADR-0012](../adr/ADR-0012-voice-profile-and-calibration.md) propagates from the manifest through
the miner's filtering into this input, and is never re-litigated here.

**Every hook carries evidence.** `evidence_refs` must point at something in the candidate — a commit,
a PR, a benchmark run, an ADR. An angle the repository does not support is the beginning of a false
post, and catching it here costs one cheap call instead of a full pipeline and a human's attention.

**Aim at the audience, not at the technology.** The hook is judged against `program.audience`, which
is the one interview answer with no default ([interview](../orchestrator/interview.md) Q0.2)
precisely because everything downstream aims at it.

**Do not write.** The output is a decision plus a one-sentence hook, never a draft. Handing the
Copywriter a half-written post collapses two stages into one and makes both unmeasurable.

## Declining

`no_angle` is a **success**, not a failure, and the pipeline is designed around it: the campaign
closes, and the cost stops at one cheap call.

Return it when the candidate scored well mechanically but carries no defensible story — a large
refactor with no decision behind it, a fix too trivial to be interesting, a change whose interesting
part is under embargo, or a story the knowledge base shows was told three weeks ago.

**A pipeline with no early exit does not save money by using a cheap model first.** The saving comes
from stopping here, not from the price per token.

## Failure modes

| Situation | Behaviour |
|---|---|
| No archetype fits | `no_angle` with a reason |
| Every eligible archetype is at its monthly ceiling | `no_angle`; the ceiling is not negotiable |
| The candidate's interesting content is outside `safety.path_allowlist` | `no_angle`; the gate already stripped it and an angle built on what remains would be hollow |
| Output fails schema validation | Retry once, then surface the stage failure |

## Verification

Archetype accuracy is scored in the [miner suite](../eval/miner.md) (≥ 0.75), because the label —
*which shape should this commit be told as* — belongs to the same corpus.

**Baseline:** always pick the most frequent archetype. Angle must beat it clearly, or the stage is
paying for a constant.

A second metric worth tracking: **the `no_angle` rate**. Near zero means it never declines, which
means the early exit is not working and every mined candidate is being paid for downstream.
