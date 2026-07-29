# `/eadros retract` — the wrong-post runbook

Withdraws or corrects published content. Owned by the **publisher** role, executed by a human.

This command exists so that the runbook is written **before** the incident rather than during it.
A maintainer who discovers at 23:00 that a post quotes a customer name should be reading a
procedure, not inventing one — and should not be discovering only then that on one of their
channels removal is impossible.

## Procedure

1. **Pause first.** Run `/eadros pause` before anything else. A wrong post usually means the gate
   missed a class of problem, and the queue behind it may carry more of the same. Stopping the
   bleeding costs nothing and is reversible; publishing three more while investigating is not.

2. **Classify the harm**, because it sets the speed/care trade-off:

   | Class | Examples | Response |
   |---|---|---|
   | `leak` | Secret, customer name, internal host, embargoed work | **Immediate**, before accuracy review |
   | `false` | A claim that is wrong | Correct or withdraw; a visible correction is usually better than a deletion |
   | `voice` | Off-tone, buzzword slipped through | Edit in place where supported; rarely worth a deletion |
   | `duplicate` | Idempotency failed | Remove the later copy |

   A `leak` is the only class where speed outranks deliberation. For everything else, a deletion
   that people noticed is worse than a correction they can follow — developers are more forgiving of
   a fixed mistake than of a disappeared one.

3. **Branch on the channel's `api.delete`.** The field is required on every channel profile
   ([ADR-0011](../adr/ADR-0011-channel-capability-tiers.md)) precisely so this branch is knowable in
   advance:

   - **`supported`** — propose the deletion, show what will be removed, delete on human
     confirmation.
   - **`edit-only`** — unpublish or correct in place. Dev.to is the standing example: an article can
     be unpublished and edited, but the URL and any syndicated copies persist.
   - **`unsupported`** — **say plainly that removal is impossible.** Hacker News is the standing
     example: a submission cannot be withdrawn. The remedy is a correction comment from the
     submitting account plus fixing the artifact the post pointed at — the README or docs page —
     because that is what a reader arriving later actually reads. Do not imply a takedown is
     pending.

4. **Fix the source of truth, not only the post.** If the post described the repository
   incorrectly, the repository is now the more-read copy of the error. A retraction that leaves the
   README wrong has moved the problem rather than solved it.

5. **Record the retraction.** Class, channel, what was done, what could not be done, and the
   original gate verdict — which passed this content and should not have.

6. **Feed the eval corpus.** Every retraction becomes a test case in `/eadros eval`. A `leak` that
   reached publication is a **gate defect**, not bad luck: the missing pattern goes into
   `safety.deny_terms` or the secret-scanning corpus, and the regression test asserts that this
   exact content would now be blocked. This is the only mechanism by which the system gets safer
   from being wrong.

7. **Review before resuming.** `/eadros resume` is the kill-switch owner's call, after step 6. The
   queue that was paused in step 1 is re-gated with the updated rules rather than released
   unchanged — otherwise the fix arrives after the queue that motivated it.

## Boundary

The agent **proposes each step and executes only on confirmation**. It never deletes published
content on its own initiative, never deletes without showing what will be removed, and never
reports a retraction as complete when the channel only permitted a correction. Where removal is
impossible it says so in those words — an honest *"this cannot be taken down"* is more useful to a
maintainer than a hopeful one.

## Flags

| Flag | Effect |
|---|---|
| `--id <id>` | The post to retract, from the URL ledger |
| `--class <c>` | `leak` / `false` / `voice` / `duplicate` |
| `--all-channels` | Retract every copy of a campaign, branching per channel |
| `--dry-run` | Show the plan per channel, including what cannot be undone, and stop |
