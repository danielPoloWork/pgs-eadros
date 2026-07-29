# EADROS State Machine

[ADR-0003](../adr/ADR-0003-agent-orchestration.md) specifies *"a state machine control"* and then
draws none. This document is that machine.

Two decisions frame it:

**The orchestrator is a fixed DAG with models inside the nodes, not an agent that plans.** The
sequence of stages is known in advance and never varies by story; what varies is the content each
stage produces. Letting a model choose the pipeline would add non-determinism, unbounded cost and
an unauditable trail to a system whose entire value proposition is governance. Where a decision is
genuinely open — *which* archetype, *which* angle — that is a model's output feeding a fixed node,
not a model choosing the graph.

**The campaign state is derived; the post state is real.** One story published to four channels is
one campaign and four posts, each with its own gate verdict, approval, quota check and outcome.
Modelling this as a single status is what makes a partial publish — two channels succeeded, one
failed, one is awaiting a human — impossible to represent. Almost every interesting failure in this
system is a partial one.

---

## Post lifecycle

```
                          ┌──────────────► rejected ◄──────────┐
                          │                                     │
  drafted ──► gated ──► pending_approval ──► approved ──► queued ──► dispatching ──► published
     │          │              │                                        │              │
     │          ▼              ▼                                        ▼              ▼
     │       blocked        expired                                  failed        retracted
     │                                                                  │
     └──► superseded                          awaiting_human_post ◄──────┘ (tier: draft)
                                                       │
                                                       └──────────────► published
```

| State | Meaning | Terminal |
|---|---|---|
| `drafted` | The copywriter produced text for this channel and locale | |
| `gated` | The output pass of the pre-publish gate passed ([ADR-0014](../adr/ADR-0014-deterministic-pre-publish-gate.md)) | |
| `blocked` | A gate stage blocked it; `gate_verdicts` holds the stage and span | ✓ |
| `pending_approval` | In the human review queue | |
| `expired` | Sat in the queue past the staleness window | ✓ |
| `rejected` | A human rejected it, with a reason | ✓ |
| `approved` | A valid approval record exists | |
| `queued` | Approved, waiting for the schedule window or a quota reset | |
| `dispatching` | Outbox intent committed; the platform call is in flight | |
| `awaiting_human_post` | `draft` tier: payload and composer link handed to a human | |
| `published` | Live, with `external_url` recorded | |
| `failed` | Retries exhausted; dead-lettered | ✓ |
| `retracted` | Withdrawn or corrected ([`retract`](../commands/retract.md)) | ✓ |
| `superseded` | A later campaign covers the same story better | ✓ |

## Transitions

| From → To | Trigger | Guard | Who |
|---|---|---|---|
| `drafted → gated` | pipeline | all ADR-0014 output stages pass | agent |
| `drafted → blocked` | pipeline | any stage blocks | agent |
| `gated → pending_approval` | pipeline | — | agent |
| `pending_approval → approved` | `/eadros approve` | a valid `approvals` row exists | **human only** |
| `pending_approval → rejected` | `/eadros reject` | reason present | **human only** |
| `pending_approval → expired` | scheduler | age > staleness window | system |
| `approved → queued` | `/eadros publish` | not paused | agent |
| `queued → dispatching` | `/eadros publish` | **not paused** ∧ tier ≠ `draft` ∧ quota remains ∧ inside window ∧ idempotency key reserved | agent |
| `queued → awaiting_human_post` | `/eadros publish` | tier = `draft` | agent |
| `dispatching → published` | platform 2xx | outcome committed | agent |
| `dispatching → failed` | retries exhausted | dead-lettered | agent |
| `awaiting_human_post → published` | human pastes the URL | `external_url` present | **human only** |
| `published → retracted` | `/eadros retract` | human confirmation per step | **human only** |
| `* → superseded` | `/eadros mine` | dedup verdict `supersedes` | agent |

Three transitions are marked **human only**, and no agent path exists to any of them. That is the
product boundary expressed as a graph: `approve`, the manual post, and `retract` are the points
where a person is accountable, and an implementation that lets an agent reach them has removed the
thing being sold.

## Guards worth stating

- **`paused` is checked at `queued → dispatching`, not earlier.** Checking only at the start of a
  run leaves a window in which a pause during a multi-channel dispatch is ignored — precisely the
  moment a kill switch is being used.
- **The idempotency key is reserved *before* the platform call**, as part of the same transaction
  that commits the outbox intent. Reserving after the call is how a timeout becomes a double post.
- **`expired` is a real state, not a silent publish.** A post about a commit from three weeks ago is
  not news, and shipping it late is worse than not shipping it. Expiry is reported, never inferred.
- **Reviewer iteration is bounded at the node, not the graph.** `drafted` may be re-entered up to
  `budget.max_reviewer_iterations` (default 2, [ADR-0013](../adr/ADR-0013-cost-control-and-model-routing.md));
  on exhaustion the post advances to `pending_approval` carrying the reviewer's open objections,
  rather than looping. An unbounded critique loop is this architecture's classic cost failure.

## Campaign state is derived

`campaigns.state` is a projection over its posts and is never written independently:

| Campaign state | Rule |
|---|---|
| `drafting` | any post in `drafted` or `gated` |
| `in_review` | any post in `pending_approval` |
| `partially_published` | ≥1 post `published` **and** ≥1 post not terminal |
| `published` | all posts terminal, ≥1 `published` |
| `abandoned` | all posts terminal, none `published` |

`partially_published` exists because it is the normal outcome, not an error: a campaign that went
live on Dev.to, is awaiting a human on Hacker News, and failed on LinkedIn is one row honestly
described. **There is no compensating unpublish.** Each post was approved for its channel
individually, and withdrawing a good post because another platform timed out would be a
self-inflicted incident ([`publish`](../commands/publish.md)).

## The transition ledger

Every transition appends to `post_transitions` — `{post_id, from, to, trigger, actor, at,
guard_results, correlation_id}` — written in the **same transaction** as the state change
([ADR-0016](../adr/ADR-0016-local-first-single-file-store.md)). The pattern is EADOS's
`delivery_state.checkpoints`: a legal, contiguous chain ending at the current state, where a
human-gated move carries the actor who confirmed it *and* the guard results that were evaluated —
evidence, not honour.

State is derived from the ledger rather than overwritten on top of it. A `posts.state` column that
disagrees with its own history is a bug the schema should surface, not a discrepancy to reconcile
by hand.

## Property tests

These are the invariants the schema cannot express, and they are the test suite that makes the
governance claim checkable rather than aspirational:

1. **No post reaches `published` without an `approvals` row** — over every interleaving of
   transitions, including crashes between outbox intent and outcome.
2. **No post enters `dispatching` while `governance.paused`** — including a pause raised mid-run.
3. **A `draft`-tier post never enters `dispatching`**, under any input, ever.
4. **Every transition chain is contiguous and legal** — no state is reachable except through a
   declared edge.
5. **A crash between intent and outcome never produces two posts** — kill the process at every
   point in the dispatch path; `--reconcile` must converge on exactly one.
6. **Reviewer iterations never exceed the configured cap**, including on repeated failures.

Tests 1–3 and 5 are the ones a reviewer should ask to see first: they are the mechanical form of
"a human approved this", "the kill switch works", "we do not post where we are not allowed to", and
"we never double-post" — the four claims the rest of the specification rests on.
