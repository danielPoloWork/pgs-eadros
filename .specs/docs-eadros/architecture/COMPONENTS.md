# EADROS Architecture: Components (C4 Level 3)

**Three of the nine pipeline components hold a model.** That is the single most important fact about
this architecture, and it is a deliberate result rather than an accident of scope.

[ADR-0003](../adr/ADR-0003-agent-orchestration.md) proposed six specialised agents. Working through
the cost model ([ADR-0013](../adr/ADR-0013-cost-control-and-model-routing.md)) and the safety model
([ADR-0014](../adr/ADR-0014-deterministic-pre-publish-gate.md)) moved four of them into
deterministic code, because in each case the job turned out to be one a model does worse, more
expensively, and untestably:

| Originally proposed | Now | Why it moved |
|---|---|---|
| Story Finder **Agent** | Deterministic scorer | Ranking with weighted signals is exact, free, and testable against a golden set. A model here would put spend on the highest-volume stage |
| Publisher **Agent** | Publisher + outbox | Dispatch is a transaction with an idempotency key. Nothing about it benefits from judgment |
| Analytics **Agent** | Deterministic aggregation | The learning loop is advisory, not closed ([ADR-0015](../adr/ADR-0015-attribution-methodology.md)); the work is arithmetic over a time series |
| Planner **Agent** | Scheduler | Cadence is decided by channel ceilings, the schedule window and campaign preconditions — all config. The undefined "80/20 ratio" it was to manage is dropped |

What is left in the models is where judgment genuinely lives: **framing, writing, and critique.**

## The pipeline

```
  repository activity
        │
        ▼
  ┌──────────────────┐   ⚙ deterministic — no model call, ever
  │ 1 Story miner    │     scores signals, applies hard exclusions, dedups
  └────────┬─────────┘
           │ top-K candidates only  ◄── the whole cost bound (ADR-0013)
           ▼
  ┌──────────────────┐   ⚙ deterministic, pass 1 of 2
  │ 2 PrePublishGate │     secrets · deny_terms · paths · embargo · taint
  └────────┬─────────┘     runs BEFORE any model sees the content
           ▼
  ┌──────────────────┐   ◆ LLM · cheap tier
  │ 3 Angle          │     picks the archetype and the hook, within consent
  └────────┬─────────┘
           ▼
  ┌──────────────────┐   ◆ LLM · mid tier          ┌──────────────────┐
  │ 4 Copywriter     │◄──────────── critique ──────│ 5 Reviewer       │ ◆ LLM · strong tier
  └────────┬─────────┘   max 2 iterations, hard    └──────────────────┘
           ▼
  ┌──────────────────┐   ⚙ deterministic, pass 2 of 2
  │ 2 PrePublishGate │     + claims resolution · voice lint
  └────────┬─────────┘
           ▼
  ┌──────────────────┐   ⚙ human gate — no agent path exists
  │ 6 Review queue   │     approval invalid unless the text changed
  └────────┬─────────┘
           ▼
  ┌──────────────────┐   ⚙ deterministic
  │ 7 Scheduler      │     channel ceilings · schedule window · preconditions
  └────────┬─────────┘
           ▼
  ┌──────────────────┐   ⚙ deterministic
  │ 8 Publisher      │     tier branch · outbox · idempotency · reconcile
  └────────┬─────────┘
           ▼                                        ┌──────────────────┐
     auto │ assisted │ draft ──► human posts        │ 9 Metrics        │ ⚙ deterministic
                                                    │   daily snapshot │
  ┌──────────────────┐  supporting, not in the DAG  └──────────────────┘
  │ Knowledge base   │  FTS5 + embeddings; read by 3, 4, 5 and by the miner's dedup
  └──────────────────┘
```

## Components

| # | Component | Kind | Input → Output | Tier | Reference |
|---|---|---|---|---|---|
| 1 | **Story miner** | ⚙ | repository activity → ranked candidates | — | [mine](../commands/mine.md) |
| 2 | **PrePublishGate** | ⚙ | candidate / draft → verdict per stage | — | [ADR-0014](../adr/ADR-0014-deterministic-pre-publish-gate.md) |
| 3 | **Angle** | ◆ | candidate + KB → archetype + hook | cheap | [ADR-0006](../adr/ADR-0006-content-generation-pipeline.md) |
| 4 | **Copywriter** | ◆ | angle + voice profile + channel profile → draft | mid | [ADR-0012](../adr/ADR-0012-voice-profile-and-calibration.md) |
| 5 | **Reviewer** | ◆ | draft → per-class defect verdicts | strong | [eval/reviewer](../eval/reviewer.md) |
| 6 | **Review queue** | ⚙ + human | draft → approval or rejection record | — | [DATA_MODEL](DATA_MODEL.md) |
| 7 | **Scheduler** | ⚙ | approved post → queued or dispatch-ready | — | [STATE_MACHINE](STATE_MACHINE.md) |
| 8 | **Publisher** | ⚙ | approved post → live post or hand-off | — | [publish](../commands/publish.md) |
| 9 | **Metrics collector** | ⚙ | published post → daily snapshot rows | — | [ADR-0015](../adr/ADR-0015-attribution-methodology.md) |
| — | **Knowledge base** | ⚙ | repo docs + past posts → retrieved context | — | [ADR-0016](../adr/ADR-0016-local-first-single-file-store.md) |

⚙ deterministic · ◆ model-bearing

### The three model-bearing nodes

**Angle** picks how to tell the story: which archetype, which hook, aimed at `program.audience`. It
selects only from archetypes the maintainer consented to — a `postmortem: false` profile means the
option is not in the set, not that the model is asked to avoid it.

**Copywriter** writes per channel and locale, constrained by the channel profile's formatting rules
and the voice fingerprint. It **never generates code snippets**: they are extracted verbatim from the
repository at a stated SHA, which is what makes the claim resolver possible at all and removes the
largest hallucination surface as a side effect.

**Reviewer** gets the strong tier because it is the last mechanical check before a human, and human
attention is the scarcest resource in the system. It critiques against a *stated standard* — the
voice profile and the claim discipline — rather than against taste, which is what makes
[its eval](../eval/reviewer.md) possible.

### What is deliberately not a model

The four demoted components share a property: each one carries a **guarantee** rather than a
judgment, and a guarantee implemented in a prompt can only ever be an aspiration.

- The **miner** must produce identical output for identical input, or the cost model has no floor.
- The **gate** must have recall 1.0 on planted secrets. That is a test result in deterministic code
  and an unprovable claim anywhere else.
- The **publisher** must never double-post. This is a transaction, not a decision.
- The **scheduler** must never exceed a channel ceiling or publish outside the window.

## Mapping to the four-stage pipeline

[ADR-0006](../adr/ADR-0006-content-generation-pipeline.md) defines
`Mining → Angle Selection → Channel Drafting → Validation & Polish`. The stage boundaries hold; the
mapping was never stated, which is how three documents came to describe three different pipelines:

| ADR-0006 stage | Components |
|---|---|
| Mining | 1 Story miner, 2 gate pass 1 |
| Angle Selection | 3 Angle |
| Channel Drafting | 4 Copywriter |
| Validation & Polish | 5 Reviewer, 2 gate pass 2 |

Everything after Validation — the human queue, the scheduler, the publisher, the metrics collector —
sits **outside** the content pipeline ADR-0006 describes. That is the correct boundary: ADR-0006
governs how text is produced, and governance over whether it ships is a separate concern with its own
decisions ([ADR-0008](../adr/ADR-0008-human-review.md),
[ADR-0011](../adr/ADR-0011-channel-capability-tiers.md),
[ADR-0014](../adr/ADR-0014-deterministic-pre-publish-gate.md)).
