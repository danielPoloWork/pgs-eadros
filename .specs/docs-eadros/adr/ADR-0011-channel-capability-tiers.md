# ADR-0011: Channel capability tiers — the platform decides the automation level, not the maintainer

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-07-29 |
| **Supersedes** | [ADR-0007 — Channel-native social adapters](ADR-0007-social-network-adapters.md) (extends it: ADR-0007 governs *formatting* per channel; this ADR governs *whether the adapter may publish at all*) |
| **Related** | [`orchestrator/channels/_schema.md`](../orchestrator/channels/_schema.md) · [`commands/onboard.md`](../commands/onboard.md) (Phase 2) |

## Context

ADR-0007 treats every destination as an adapter that differs only in formatting: character limits,
Markdown flavour, tag syntax. That model is false in the way that matters. The destinations differ
first in **whether automated posting is permitted at all**, and only second in how the text is
shaped.

Three platforms the specification names as first-class channels do not admit an adapter in the
ADR-0007 sense:

- **Hacker News** publishes no write API, and automated submission runs against site norms.
  Critically, penalties there attach to the **domain**, not the account — a ban outlives the account
  that earned it and cannot be escaped by registering another.
- **Reddit** exposes an API, but self-promotion rules are set per subreddit and enforced by
  moderators and automation that predate any adapter we would write.
- **LinkedIn** gates programmatic posting behind app review, with credentials that expire in weeks.
  An unapproved app is not a degraded channel; it is no channel.

Meanwhile **Dev.to, Hashnode, Mastodon, Discord and GitHub** do offer documented write APIs with no
prohibition on automation.

The failure mode is asymmetric and that asymmetry is the whole decision. A tool that under-automates
costs its user some manual effort. A tool that over-automates can get the **domain of the project it
was hired to promote** banned from the single highest-signal channel in the ecosystem — an outcome
that is permanent, uncompensable, and caused by the tool itself. This is not a risk to be balanced
against convenience; it is a risk to be designed out.

A second-order problem compounds it: platform terms change, and they change without notice to us.
Any encoding of "what is allowed" that cannot be re-verified on a schedule will be silently wrong
within a year.

## Options considered

**A. Capability tiers derived from a versioned, date-stamped channel profile** *(chosen)*

Each destination gets a `channels/<name>.yaml` recording what the platform permits, with a
`policy.verified_on` date. The profile derives one of three tiers — `auto`, `assisted`, `draft` —
and the publisher branches on it.

- ✅ **The dangerous case is unrepresentable, not merely discouraged.** A `draft`-tier channel has
  no code path to dispatch. Preventing the domain ban is a property of the architecture rather than
  a rule someone must remember.
- ✅ **Compliance is data, so it can be re-verified.** `verified_on` makes staleness *visible*:
  `/eadros doctor` warns past 90 days and the interview refuses to settle Phase 2 on a stale
  profile. A rule that lives in code cannot be audited on a schedule; a rule in a dated file can.
- ✅ **`draft` is a first-class outcome, not a gap.** EADROS still does the work that is actually
  hard — picking the story, writing the title, choosing the moment, running the pre-publish
  checklist — and hands a human the last click. The returned URL closes the analytics loop, so the
  channel is fully instrumented despite never being automated.
- ✅ **Extensible on the EADOS profile pattern.** "There is no unsupported channel, only a channel
  not yet profiled" mirrors the language-profile stance exactly, so contributors already know the
  shape: copy the template, verify the terms, add an ADR.
- ❌ **Someone must actually re-verify the policies.** Mitigated by making the date a gate rather
  than a note: a stale profile blocks the interview and fails `/eadros audit`, so decay surfaces as
  a failure instead of accumulating quietly.
- ❌ **A maintainer may find `draft` paternalistic.** Mitigated by allowing the override — through
  a recorded ADR carrying their explicit acceptance, never a config flag. The point is not to
  prevent the choice; it is to prevent the choice being made *accidentally* by someone who did not
  know that HN bans propagate to the domain.

**B. Uniform adapters, every channel equal (the ADR-0007 status quo)** — ✅ simplest model, one
interface, trivial to document; ❌ it encodes a factual error about the world, and the error's
first materialisation is a permanent domain ban. It also gives the maintainer no way to learn the
constraint before violating it: the tool would look like it worked, right up until it did not.
**Rejected.**

**C. Per-channel special cases in the publisher** — ✅ no new file format, ships fastest;
❌ compliance rules end up scattered across code paths with no verification date, no audit surface
and no way for a community contributor to add a channel without touching the publisher. This is how
the rules become unverifiable, which is the exact defect that makes option B dangerous. **Rejected.**

**D. Automate everywhere, disclaim the risk in the README** — ✅ maximum apparent capability, the
feature matrix looks complete; ❌ transfers a permanent, uncompensable risk onto the user in
exchange for a checkbox, on a tool whose entire premise is protecting a project's technical
credibility. A DevRel tool that damages a project's standing has no defence. **Rejected on
principle, not on cost.**

## Decision

Every destination is described by a `channels/<name>.yaml` profile carrying a dated policy record
and a derived `tier`:

| Tier | Condition | Publisher behaviour |
|---|---|---|
| `auto` | Documented write API, automation permitted | Dispatches on human approval |
| `assisted` | Write API exists but constrained — quota, app review, expiring credentials | Dispatches under a metered budget; refuses when quota is spent |
| `draft` | No lawful automation path | **Never dispatches.** Emits payload, composer link and checklist for a human |

The tier is **stated to the maintainer at onboarding, never offered as a choice** — they decide
*whether* to use a channel; the profile decides *how*. Raising a tier requires a new ADR recording
the maintainer's explicit acceptance of the specific risk, with `ban_scope` quoted.
`policy.verified_on` older than 90 days blocks the interview and fails `/eadros audit`.

`api.delete` is a required profile field, because the retraction runbook branches on it and a
channel where retraction is impossible must be known to be strictest *before* publishing.

## Consequences

- The most damaging failure available to this product — a self-inflicted domain ban — has no code
  path. It is prevented structurally rather than by policy documentation.
- Compliance becomes an auditable, dated artifact that a community contributor can extend and a
  reviewer can check, instead of tribal knowledge embedded in adapter code.
- The feature matrix reads as smaller than a competitor's that claims to auto-post everywhere. That
  gap is the honest one, and stating *why* Hacker News is `draft` is a stronger credibility signal
  to this audience than claiming to automate it.
- `assisted` makes quota a first-class product constraint: an unapproved LinkedIn app is recorded as
  a roadmap item, not silently accepted as a channel that then drops half the weekly output.
- Every `draft` channel needs a hand-off loop and a returned URL, so the analytics engine must
  accept human-supplied post URLs as a normal input rather than an exception.
