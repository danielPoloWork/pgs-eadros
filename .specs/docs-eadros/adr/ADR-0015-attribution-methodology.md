# ADR-0015: Attribution is directional — owned redirects, daily snapshots, and no causal claim

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-07-29 |
| **Amends** | [ADR-0009 — Metrics engine](ADR-0009-metrics-engine.md) (keeps the engine; replaces its correlation claim with a method the data supports) |
| **Related** | [`vision/SUCCESS_METRICS.md`](../vision/SUCCESS_METRICS.md) · [`orchestrator/channels/_schema.md`](../orchestrator/channels/_schema.md) |

## Context

ADR-0009 specifies an engine that *"correlates published posts with GitHub Traffic API data
(clones, unique visitors, referral paths, stargazers, forks)"* and feeds the result back so the
Angle Agent can *"double down on high-performing topics."* Three facts make that unachievable as
written, and they are facts about the world rather than about our implementation:

1. **The Traffic API retains roughly 14 days**, aggregated, with a top-10 referrer list. There is
   no per-post granularity, and — decisively — **a day not captured is gone permanently.** A
   metrics engine that queries on demand is not late; it is empty.
2. **Referrers are stripped or rewritten** by the channels that matter most. In-app browsers and
   link wrappers mean the attribute the correlation depends on frequently never arrives, and the
   loss is biased: it is worst on exactly the channels with the largest audiences.
3. **The cadence makes significance unreachable.** At one substantive post per week — the ceiling
   the voice archetypes and channel profiles deliberately impose — per-angle comparison will not
   reach statistical power within any horizon a maintainer cares about. The learning loop as
   specified is not merely imprecise; at this volume it is **statistically hollow**.

The right question is therefore not *"how do we attribute"* but *"what may we truthfully claim."*
That question has extra weight here: this is a tool whose product is credibility. A DevRel system
that overstates its own metrics has falsified its pitch in its own dashboard.

## Options considered

**A. Owned redirect + UTM, mandatory daily snapshots, pre/post lift against a control window,
reported as directional** *(chosen)*

- ✅ **The snapshot fixes the only unrecoverable problem.** Everything else can be improved later;
  data not captured inside the 14-day window cannot. Making the snapshot infrastructure rather than
  a feature is the highest-value decision in this ADR.
- ✅ **An owned redirect is the only per-post attribution that survives referrer stripping.** The
  link carries the identity, so it does not matter what the platform does to the header.
- ✅ **Interrupted time series is the correct instrument for n=1 interventions.** Lift against a
  pre-window baseline, with the window stated, is a claim the data actually supports.
- ✅ **Honesty is a feature for this audience.** A dashboard that says *"LinkedIn: referrer not
  recoverable; lift measured against a 14-day baseline"* is more persuasive to a developer than a
  confident number they can tell is fabricated.
- ❌ **Weaker headline numbers than a competitor claiming causal attribution.** Accepted
  deliberately; the alternative is a number we know to be wrong.
- ❌ **A redirect is infrastructure to run.** Minimal — a static redirect on the project's existing
  docs domain, which most target projects already have.

**B. Claim causal attribution from referrer data** — ✅ the dashboard everyone wants, and the one
ADR-0009 currently promises; ❌ the data does not support it and the failure is invisible, so the
maintainer makes real content decisions on noise. Self-refuting for this product in particular.
**Rejected.**

**C. A self-hosted analytics platform** — ✅ genuinely richer data; ❌ privacy obligations,
infrastructure a solo maintainer will not run, and it **does not touch the significance problem**,
which is set by cadence rather than by instrumentation. It buys precision on a quantity that is
still underpowered. **Rejected.**

**D. Drop analytics** — ✅ intellectually clean, no false claims; ❌ the maintainer genuinely needs
to know what is landing. The answer to untrustworthy instruments is honest instruments, not none.
**Rejected.**

## Decision

1. **`/eadros metrics` runs daily, append-only**, unique on `(post_id, metric_date)`. Not
   on-demand, not optional. A missed window is recorded as a gap rather than interpolated.
2. **Every link is an owned redirect carrying UTM parameters** — except where a channel profile
   sets `attribution.utm: unsupported`. Hacker News is the standing example: a tagged Show HN link
   reads as marketing and the audience penalises it, so the clean URL is correct and the attribution
   loss is accepted and recorded.
3. **Lift, not attribution.** Effect is reported as pre/post lift against a stated control window,
   labelled with the window length and the sample size.
4. **`success.attribution` accepts `directional` only.** `causal` is not a permitted value.
5. **The learning loop is advisory, not closed.** The analytics component — deterministic
   aggregation, not an agent ([COMPONENTS](../architecture/COMPONENTS.md)) — surfaces ranked observations
   **with sample sizes, and states explicitly when n is too small to distinguish angles.** It does
   not silently update the Angle Agent's priors. A long-horizon bandit is permitted only above a
   stated volume threshold and is off by default.
6. **Report what could not be measured.** Every digest names the channels where attribution was
   lost. An absent number and a zero must never render the same.

## Consequences

- The metrics are fewer and smaller, and they are true. For a tool selling technical credibility
  that trade is the product, not a compromise.
- The daily snapshot becomes infrastructure with an availability requirement — the one job whose
  failure destroys data rather than delaying it — so `/eadros doctor` and `/eadros audit` both
  check its recency.
- `SUCCESS_METRICS.md` could not survive unchanged, and was rewritten accordingly: *"Channel-Native
  Resonance Score, normalised for developer audience density"* is not computable, because audience
  density is not observable. It was **removed** rather than redefined, along with "Weekly Active
  Users" and "returning visitors ratio" — neither of which GitHub exposes.
- The Angle Agent's feedback becomes a human-reviewed dashboard rather than an autonomous loop,
  which also removes an unbounded cost path ([ADR-0013](ADR-0013-cost-control-and-model-routing.md)).
- Every channel profile must declare `attribution.referrer_preserved`, since the analytics engine's
  honesty depends on knowing where its data is blind.
