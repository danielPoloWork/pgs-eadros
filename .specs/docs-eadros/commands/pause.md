# `/eadros pause` · `resume` — the kill switch

Stops every dispatch, immediately.

## `pause`

Sets `governance.paused: true` and records who and why. Takes effect at the
`queued → dispatching` guard, which is checked **per dispatch, not per run**
([STATE_MACHINE](../architecture/STATE_MACHINE.md)) — a pause raised during a multi-channel campaign
stops the channels that have not gone yet.

**Anyone may pause.** No permission check, no confirmation prompt, no reason required — though one is
recorded when given. A kill switch that asks a question is a kill switch that gets used too late, and
the cost of an unnecessary pause is a delayed post.

What pause does **not** do: it does not retract what is already live ([retract](retract.md)), and it
does not stop mining or drafting. Those spend money but publish nothing, and stopping them is
[`budget.on_exceed`](../adr/ADR-0013-cost-control-and-model-routing.md)'s job, not this one.

Everything queued survives. Pause is a hold, not a discard.

## `resume`

**Only `governance.kill_switch_owner` may resume.** Asymmetry is the point: stopping is cheap and
should be frictionless; restarting is a decision about whether the reason for stopping is resolved.

Resume does not simply release the queue. It **re-gates every held post** under the current rules,
because a pause is usually followed by a fix — a new deny-term, a corrected voice profile, a tightened
allowlist — and releasing the queue unchanged would ship the content that motivated the pause. The
fix must reach the queue that caused it.

Posts that now fail the gate move to `blocked`, and are reported by id rather than counted.

## Typical sequence

```
1. something is wrong          → /eadros pause
2. withdraw or correct         → /eadros retract --id <id> --class leak
3. fix the cause               → deny-term, allowlist, or voice profile
4. write the regression test   → /eadros eval --add-from-retractions
5. release, re-gated           → /eadros resume
```

Step 4 before step 5 is the ordering that matters. Resuming before the failure became a test means the
next occurrence is discovered the same way as this one — by a reader.

## Boundary

`pause` writes one manifest field and takes effect immediately. `resume` is owner-only and re-gates
before releasing. Neither publishes, retracts or deletes anything.

## Flags

| Flag | Effect |
|---|---|
| `--reason "..."` | Recorded with the pause; shown in every `status` until resumed |
| `--dry-run` (on `resume`) | Show which held posts would pass or fail the current gate, and release nothing |
