# `/eadros draft <candidate-id>` — run the pipeline for one story

Takes one mined candidate through Angle → Copywriter → Reviewer to the review queue. Owned by the
**orchestrator**.

The one command that spends money. Everything before it is free; everything after it is governance.

## Procedure

1. **Budget check, before anything.** Estimate the run against `budget.cost_per_week` minus spend so
   far. Refuse under `--budget-check` when it would exceed; warn otherwise. A run that discovers the
   ceiling halfway leaves a campaign half-drafted and the ledger harder to read.

2. **Open the campaign.** `campaign.opened.v1`, carrying the correlation id that every subsequent
   event in this story's life will share.

3. **Angle** (cheap tier). Receives the candidate, the audience, the positioning and **only the
   consented archetype set**. May return `no_angle` — a success, which closes the campaign after one
   cheap call ([agents/angle](../agents/angle.md)).

4. **Extract snippets.** Verbatim, from the repository at the candidate's SHA, filtered by
   `safety.path_allowlist` and capped at `safety.max_diff_lines`. **This happens before the
   Copywriter runs**, because the Copywriter never generates code — it selects from what was
   extracted.

5. **Copywriter** (mid tier), once per `(channel, locale)`. Constrained by the channel profile and
   the voice fingerprint. Emits `claims[]` alongside the body.

6. **Resolve claims deterministically.** Each `source_ref` is checked: does the SHA exist, does
   `file:line` exist at that SHA, does the number match the benchmark record. Unresolved claims are
   attached to the draft as findings; they are not softened.

7. **Reviewer** (strong tier). Critiques against the stated standard and hunts the assertions the
   Copywriter never registered — the resolver's structural blind spot
   ([agents/reviewer](../agents/reviewer.md)). Loop back to step 5 at most
   `budget.max_reviewer_iterations` times.

8. **Gate, output pass.** All eight stages over the final text
   ([ADR-0014](../adr/ADR-0014-deterministic-pre-publish-gate.md)). A block ends the draft's life at
   `blocked`, with the stage and span recorded.

9. **Queue for review.** `review.requested.v1`, with an expiry.

10. **Record.** Model, prompt version, tokens and cost per stage into `budget_ledger`.

## Failure behaviour

| Stage outcome | Result |
|---|---|
| `no_angle` | Campaign closes. One cheap call spent — the intended cheap exit |
| Iteration cap reached | Draft advances to the human **with open objections attached** |
| Gate blocks | State `blocked`; the verdict names the stage and span; no human is asked to look at it |
| Schema invalid | Retry once, then surface the stage failure rather than parsing prose |
| Budget ceiling crossed mid-run | Finish the current channel, queue the rest, emit `budget.exceeded.v1` |

Finishing the current channel rather than aborting mid-draft is deliberate: a half-generated draft is
waste already paid for, and abandoning it saves nothing.

## Boundary

Drafts only. `draft` never approves, never schedules and never publishes — its terminal state is a
post sitting in the review queue, or a campaign that closed without one.

## Flags

| Flag | Effect |
|---|---|
| `--explain` | Print the routed tier, prompt version and cost estimate per stage **before** spending |
| `--budget-check` | Refuse to start if the run would exceed the weekly ceiling |
| `--channel <name>` | Draft for one channel instead of all configured |
| `--dry-run` | Run against a mock provider; no spend, no queue entry |
