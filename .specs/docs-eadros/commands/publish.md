# `/eadros publish` — dispatch approved content, within what each channel permits

Dispatches human-approved content. Owned by the **publisher** role.

This is the only command that talks to a platform, and the only one whose mistakes are public. Its
design assumption is therefore that **it will fail** — mid-run, mid-channel, with a token that
expired an hour ago — and that a failure must never resolve into a double post.

## Order of checks

The sequence is normative. Each gate is cheaper than the one after it, and each protects against a
failure the next one cannot undo.

1. **Paused?** `governance.paused` blocks everything, immediately and without exception. The kill
   switch is worthless if it is checked third.
2. **Approval record present and valid?** No approval, no dispatch. Under
   `approval_mode: edit-required` an approval whose final text is byte-identical to the generated
   draft **is not valid** unless `--as-is` was passed, and `--as-is` is recorded in the audit trail
   with the approver's identity. The agent may never create an approval record.
3. **Pre-publish gate, output pass.** The full
   [ADR-0014](../adr/ADR-0014-deterministic-pre-publish-gate.md) stage list over the *final* text —
   the approved text, not the generated one, because a human edit can reintroduce what the gate
   removed. Fails closed.
4. **Tier.** The branch that decides whether this command publishes at all
   ([ADR-0011](../adr/ADR-0011-channel-capability-tiers.md)).
5. **Quota and window.** `assisted` channels check remaining quota; every channel checks
   `schedule.window`. Outside the window the item is **queued, not dropped**.
6. **Idempotency.** See below.
7. **Dispatch, then record.**

## Tier behaviour

**`auto`** — dispatch on approval. Set `canonical_url` to the project's own docs where the profile
supports it: syndicating without a canonical lets the platform outrank the source you are trying to
grow, which inverts the point of publishing.

**`assisted`** — dispatch under a metered budget. **Refuse when quota is spent**; never borrow from
next month. A channel whose app is `pending` is not publishable, and attempting it is an error
rather than a silent skip — a weekly cadence that quietly ships half its output is worse than one
that stops and says so.

**`draft`** — **never dispatches.** Emits the payload, the pre-filled composer URL from the
profile's `handoff.url_template`, and the channel checklist. Marks the item `awaiting_human_post`
and waits. When the human pastes the live URL back, record it and hand off to `/eadros metrics` —
the channel is fully instrumented despite never being automated, which is the whole argument for
the tier existing.

## Idempotency and partial failure

Every dispatch carries an **idempotency key of `(campaign_id, channel)`** and goes through an
**outbox**: the intent is committed before the call, and the outcome is committed after it. A run
interrupted between the two resumes by *reconciling* — asking the platform whether the post exists —
never by retrying blind.

This is not ceremony. A double post is public, embarrassing, and on some channels
[unretractable](retract.md). It is the failure this command exists to prevent, and it is the one
that a naive retry loop produces on the first network timeout.

**Channels are independent.** A campaign publishing to four channels that succeeds on two is
recorded as two successes and two retriable failures. There is **no compensating unpublish** of the
successful ones: the content was approved for each channel individually, and retracting a good post
because a different platform timed out would be a self-inflicted incident. Retries use exponential
backoff with jitter; exhausted retries go to a dead-letter queue and surface in `/eadros status`.

## What is recorded

Per dispatch: `external_url`, `published_at`, the approver and approval mode, the resolved
idempotency key, the gate verdict, and the model/prompt/cost attribution carried from the pipeline
([ADR-0013](../adr/ADR-0013-cost-control-and-model-routing.md)). The URL ledger is what
`/eadros metrics` and `/eadros retract` both read; an unrecorded post is one that cannot be measured
and cannot be withdrawn.

## Boundary

The agent dispatches **only** what a human approved, **only** where the tier permits, and **only**
inside the schedule window. It cannot approve, cannot raise a tier, cannot exceed a quota, and
cannot publish while paused. Everything it does is recorded before it does it.

## Flags

| Flag | Effect |
|---|---|
| `--dry-run` | Full path against a mock publisher. **The default for a program's first week** |
| `--id <id>` | Publish one approved item rather than the queue |
| `--as-is` | Permit an unedited approval. Recorded with the approver's identity, never silent |
| `--reconcile` | Query platforms for the outbox's in-flight intents and settle them |
