# `/eadros audit` — governance audit

Asks whether the controls are still working. [`doctor`](doctor.md) checks whether the system *can*
run; `audit` checks whether it has been running **honestly**.

Every control in this specification degrades in a way its own metrics do not show. This command is
where that degradation becomes visible.

## Checks

### The gate is still trusted

| Metric | Threshold | What a breach means |
|---|---|---|
| Gate false-positive rate | ≤ 0.10 | The gate is blocking good content and is on its way to being switched off |
| Blocks by stage, trend | — | A stage suddenly dominating usually means a bad deny-term, not a new threat |
| Retraction rate, class `leak` | **0** | A gate defect. Owes a regression test ([eval/adversarial](../eval/adversarial.md)) |

**An unmeasured gate is a gate on its way to being disabled.** A rising false-positive rate is a
defect *in the gate*, not in the maintainer who started overriding it.

### The human gate is still human

This is the check the rest of the system cannot perform on itself:

| Signal | Healthy | Degraded |
|---|---|---|
| `--as-is` approval share | low | rising — approvals are becoming stamps |
| Maintainer edit distance | falling | **falling to near zero** |
| Rejection rate | 0.10 – 0.40 | below 0.10 |
| Median review time | ≥ 60 s | seconds — nobody is reading |

**Edit distance falling to zero is ambiguous, and only the rejection rate disambiguates it.** Falling
edits *with* a healthy rejection rate is the voice profile converging — the calibration loop working
as designed. Falling edits *with* a collapsing rejection rate is
[ADR-0014](../adr/ADR-0014-deterministic-pre-publish-gate.md)'s decay curve arriving on schedule.
Neither metric alone can tell you which, which is why this report always shows them together.

### Tier compliance

- Every published post's `tier_at_publish` matched its profile **at the time**.
- No `draft`-tier post ever reached `dispatching`.
- No `assisted` channel exceeded its quota.
- Any tier override has a recorded ADR.
- No channel policy is older than 90 days.

### Budget

Spend against ceiling, **per stage**. A rising drafting-stage share with flat quality metrics is the
context-growth regression that no quality suite detects ([eval/cost](../eval/cost.md)).

Also: prompt-cache hit rate, reviewer iterations distribution, and cost per published post against
its target.

### Measurement integrity

- Snapshot gap rate ≤ 1% of days.
- Any post whose metrics were never captured.
- Any campaign with no precondition ([campaign](campaign.md)).
- Any report that stated a number where a gap existed — an absent measurement rendering as a zero is
  a reporting defect, not a data one.

## Output

Findings by severity, each with the control it belongs to and the change that would resolve it.
Trends over the last four periods where a trend is the point — a false-positive rate of 0.08 is fine;
0.08 after 0.03, 0.05, 0.06 is a direction.

## Boundary

Read-only. Changes no thresholds, no manifest values, no prompts. **An audit that tunes what it
measures is not an audit** — findings go to the maintainer, who decides.

## Flags

| Flag | Effect |
|---|---|
| `--period <n>` | Periods to cover (default 4) |
| `--section <gate\|human\|tiers\|budget\|measurement>` | One section |
| `--json` | Machine-readable |
