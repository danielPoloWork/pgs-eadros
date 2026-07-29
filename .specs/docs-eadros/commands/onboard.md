# `/eadros onboard` — settle channels, voice, safety and budget

The command that makes a framed program operable. It runs interview
[Phases 2–5](../orchestrator/interview.md) and ends with a **voice calibration loop** against a
real commit from the repository. Owned by the **devrel-architect** role.

Until `onboard` completes, `mine`, `draft`, `weekly` and `publish` all refuse to run. That is
deliberate: the pipeline has no safe default for *which channels may publish*, *what may be
quoted from a private repository*, or *what a week is allowed to cost*.

## Procedure

1. **Channels (Phase 2).** Offer the profiled set from
   [`channels/`](../orchestrator/channels/_schema.md), seeded with whatever EADOS Q4.8 already
   recorded.

   **Check `verified_on` before the phase settles.** A channel profile whose policy was last
   verified more than 90 days ago must be re-verified or flagged; platform terms change under you,
   and a stale tier assignment is how an account gets banned by a rule that was added last quarter.

   **Never offer the tier as a choice.** The maintainer chooses *whether* to use a channel; the
   profile decides *how*. State each tier with its one-line reason:

   > *"Dev.to: `auto` — documented write API, automation permitted. LinkedIn: `assisted` — needs an
   > approved app, credentials expire in 60 days. Hacker News: `draft` — no write API exists, so we
   > prepare the submission and you post it."*

   A maintainer asking to raise a `draft` tier is asking to risk a **ban that attaches to the
   domain and outlives the account**. Refuse the silent override; require a recorded ADR carrying
   their explicit acceptance. Their project, their call — but with a paper trail, not a flag.

   Record an unapproved platform app as `app_status: pending` and treat it as a **roadmap item, not
   a channel**. A pending app recorded as live is how a weekly cadence silently ships half its
   output.

2. **Voice (Phase 3).** Ask for **two or three things the maintainer has actually written**.
   Extract the fingerprint into `voices/<slug>.yaml`
   ([schema](../orchestrator/voices/_schema.md)), record every derived field as
   `inferred_from_sample`, and **show it back in plain language before adopting it**.

   If no sample exists, seed from a shipped profile and **say which, explicitly**. A seeded voice
   presented as the maintainer's own is the one failure this whole phase exists to prevent.

   Then ask the archetype consent question. `postmortem` and `opinion` default **off** and must be
   asked: publishing a failure story or a stated opinion under someone's name is a reputational act
   they did not authorise, and a tool that takes it once will not be trusted again.

3. **The calibration loop — the phase ends on a sample, not an answer.**
   - Take **one real recent commit** from the repository.
   - Generate **three short drafts** under the captured profile.
   - The maintainer picks one and **edits it freely**.
   - **Diff the edit against your draft** and fold the delta into the fingerprint.

   **What they cut is worth more than what they kept.** Deletions expose the constructions the
   voice rejects — precisely what `forbidden.constructions` needs, and what no amount of asking
   produces. Append the round to `voice.calibration[]`.

   Two rounds is normal. **Converging on the first round is suspicious** — say so, and offer a
   second: a maintainer being polite about a draft they do not love will not correct it later, they
   will simply stop using the tool.

   Re-runnable at any time: `/eadros onboard --recalibrate` runs this loop alone, and should be
   re-run whenever the maintainer finds themselves rewriting drafts heavily.

4. **Safety and budget (Phase 4).** Path allowlist, deny terms, embargo, approvers, approval
   strength, budget ceilings, kill-switch owner.

   For a **private** repository, state the deny-all starting posture rather than asking whether
   they want it: nothing is quotable until paths are opted in. State that **secret scanning runs
   unconditionally and cannot be turned off** — do not present it as a preference.

   Budget answers must be **numeric**. "Keep it reasonable" is not a ceiling, and a ceiling with no
   number fails the `/eadros audit` gate exactly as an EADOS hard NFR budget with no number does.

5. **Cadence and campaigns (Phase 5).** Lay out **all** milestone campaigns now, each with the
   pre-condition that must hold before it may fire. A campaign with no pre-condition is how a
   project spends its one launch on a README that was not ready.

6. **Confirm — three separate presentations, not one wall.**
   - The **manifest**, with `defaulted` and `inferred_from_sample` entries called out by name.
   - The **tier table**, so nobody is surprised later by a hand-off they did not know existed.
   - The **first week**, concretely: what will be drafted, on which channels, at what estimated
     cost.

7. **Recommend `--dry-run` for week one.** The full pipeline against a mock publisher, so the first
   real post is not also the first test of the publisher.

## Boundary

The agent **drafts** the manifest and **extracts** the voice fingerprint; the **human confirms**
every value, acknowledges every tier, and owns the kill switch. `onboard` contacts no platform and
publishes nothing — it only reads the repository, to write the three calibration drafts.

## Re-running

`onboard` is idempotent and re-runnable. Adding a channel, rotating a credential, or recalibrating
a voice re-enters the relevant phase alone and bumps `manifest_rev`. Every re-run re-confirms —
**never operate from an unconfirmed manifest.**
