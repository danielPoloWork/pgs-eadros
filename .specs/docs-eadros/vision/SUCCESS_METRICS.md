# EADROS Success Metrics

Two questions, deliberately kept apart because they have different answers and different audiences:

1. **Is the project growing?** — the DevRel outcome.
2. **Is EADROS doing its job?** — the tool's own operating health.

A program can be excellent at the second and failing at the first, and conflating them is how a tool
reports green while the project it serves goes nowhere.

## Reading these tables

Every metric declares **how it is obtained**, because the difference between *measured* and *guessed*
is the whole credibility of a DevRel tool ([ADR-0015](../adr/ADR-0015-attribution-methodology.md)):

| Source | Meaning |
|---|---|
| `api` | Read from a platform API. Exact |
| `redirect` | From the owned redirect. Exact, per post |
| `derived` | Computed from the local store. Exact |
| `manual` | A human observes and records it. **No API exists** |

And every outcome metric declares its **attribution class**: `measured` (the number is what it says)
or `directional` (real, but its *cause* cannot be established). Nothing here is ever `causal`.

> **The 14-day constraint.** The GitHub Traffic API retains roughly two weeks. Every `api` metric
> below depends on the daily snapshot having run; a day not captured is gone permanently, not late.
> `post_metrics.is_gap` and `repo_traffic.is_gap` record misses so an absent number never renders as
> a zero.

---

## Part 1 — Is the project growing?

### North Star, by program goal

The primary metric is not universal — it follows `program.goal` from the intake interview, because
a repository built as a hiring artifact and one built for adoption are not succeeding at the same
thing:

| `program.goal` | North Star | Source | Why this one |
|---|---|---|---|
| `adoption` | **Unique cloners / week** | `api` | Someone actually ran `git clone`. Far harder to game than stars, and it is an action rather than a gesture |
| `contribution` | **External PR authors / quarter** | `api` | Distinct people, not PR count — one prolific contributor is not a community |
| `credibility` | **Inbound references / quarter** | `manual` | Newsletters, talks, third-party posts. No API exists; a human records them |
| `hiring` | **Qualified inbound conversations** | `manual` | The only honest measure, and it is a count a person keeps |

Two of the four require manual observation. Saying so is better than substituting an automatable
proxy that measures something else.

### Tiered indicators

**Leading** — movement here precedes everything else.

| Metric | Source | Attribution | Baseline | Target |
|---|---|---|---|---|
| Unique cloners / week | `api` | measured | first 4 weeks | +50% by day 90 |
| Unique visitors / week | `api` | measured | first 4 weeks | +100% by day 90 |
| Click-through from published posts | `redirect` | **measured, per post** | — | ≥ 2% of impressions where impressions are reported |
| Docs entry-page views | `api`/`manual` | directional | first 4 weeks | trending up |

Click-through is the only per-post number in this document that is exact, and only because the
redirect is owned. Platform referrers are stripped or rewritten on the channels with the largest
audiences, which is why nothing else here claims per-post precision.

**Engagement & conversion** — is attention becoming use?

| Metric | Source | Attribution | Target |
|---|---|---|---|
| Fork-to-star ratio | `api` | measured | trending up — forks are intent, stars are applause |
| External PRs opened / quarter | `api` | measured | ≥ 1 by day 90 |
| Issues and discussions opened by non-maintainers | `api` | measured | ≥ 3 by day 90 |
| Post → clone lift | `redirect` + `api` | **directional** | stated with its control window, never as a cause |

**Ecosystem authority** — lagging, mostly `manual`, and slow by nature.

| Metric | Source | Target |
|---|---|---|
| Newsletter or aggregator inclusions | `manual` | ≥ 1 by day 180 |
| Third-party posts, talks or videos referencing the project | `manual` | ≥ 1 by day 180 |
| Awesome-list inclusions | `manual` | ≥ 1 by day 180 |
| Distinct contributors, cumulative | `api` | ≥ 3 by day 365 |

### On stars

Stars are recorded because they are free to collect and readers use them as a signal. **They are
never a target and never a North Star** — a metric that rises when a post does well on an aggregator
and does not move when someone adopts the project is measuring reach, not value.

---

## Part 2 — Is EADROS doing its job?

All `derived` from the local store, therefore exact. These are the numbers that decide whether the
tool stays installed.

| Metric | Definition | Target |
|---|---|---|
| **Maintainer time per published post** | Median minutes in the review queue | ≤ **5 min** |
| **Story mining precision** | posts published ÷ candidates drafted | ≥ **0.60** |
| **Review-to-publish cycle time** | Median hours, `campaign.opened` → `post.published` | ≤ **48 h** |
| **Cost per published post** | `budget_ledger` total ÷ posts published | ≤ **€0.15** |
| **Gate false-positive rate** | verdicts marked `false_positive` ÷ blocks | ≤ **0.10** |
| **Retraction rate** | posts retracted ÷ published | ≤ **0.02**, and **0** for class `leak` |
| **Rejection rate** | drafts rejected ÷ drafts reviewed | 0.10 – 0.40 (see below) |
| **Maintainer edit distance** | Diff ratio, `approvals.hash_before` → `hash_after` | falling, **but not to zero** |

Four of these deserve their reasoning stated, because the naive target is wrong.

**Maintainer time per published post is the tool's real success metric.** The problem statement is a
time bottleneck; if EADROS costs a maintainer more attention than writing the post themselves would
have, every other number here is irrelevant. It is measured from queue timestamps, not estimated.

**Edit distance falling toward zero is a warning, not a win.** It means either the voice profile has
converged or the maintainer has started rubber-stamping — and those look identical in this metric
alone. **Read it against the rejection rate:** falling edits *with* a healthy rejection rate is
calibration working; falling edits *with* a collapsing rejection rate is
[ADR-0014](../adr/ADR-0014-deterministic-pre-publish-gate.md)'s decay curve, arriving on schedule.

**Rejection rate is bounded on both sides.** Below 0.10 the human gate has stopped functioning as a
gate. Above 0.40 the pipeline is wasting money generating content nobody wants, and the fix is
upstream in the miner or the voice profile, not in the reviewer.

**Retraction rate for class `leak` is zero, not low.** Every other target here is a trade-off; this
one is a defect count, and a single occurrence is a gate failure that owes a regression test
([eval/adversarial](../eval/adversarial.md)).

---

## Removed from this document

**"Channel-Native Resonance Score — engagement rate per channel normalised for developer audience
density."** Removed rather than restated: **audience density is not observable.** No platform
publishes the developer share of a post's reach, and no proxy for it survives contact with the
channels that strip referrers. A composite metric built on an unmeasurable denominator produces a
number that looks precise and means nothing — which, in a tool whose product is technical
credibility, is worse than having no metric at all.

The question it was reaching for — *which channel is worth the effort?* — is answered instead by
click-through from the owned redirect (`measured`) alongside cost and maintainer time per post
(`derived`). Three exact numbers, rather than one invented one.

**"Weekly Active Users."** A repository has no sessions and no users to count. Replaced by unique
cloners and unique visitors, which are real and available.

**"Returning visitors ratio."** The Traffic API reports unique visitors, not returning ones. There is
no honest way to compute this from what GitHub exposes.
