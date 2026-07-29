# EADROS Roadmap

| | |
|---|---|
| **Version** | 2.0 |
| **Date** | 2026-07-29 |
| **Status** | **Specification stage — no implementation exists** |

> **Nothing in this project is built.** Every implementation box below is unchecked, and that is the
> accurate state. What exists is the specification: 16 ADRs, the C4 views, the data model, the state
> machine, the event contract, the intake interview and the verification strategy.
>
> The `Spec` column tracks that work honestly, because it is real work and it is what this repository
> currently is. The `Impl` column tracks code, and it is empty.

**Legend** — `Spec`: ✅ specified · ◐ partial · ○ not written. `Impl`: ○ not started.

---

## M1 — Foundation

**Goal:** a manifest, a store, and a pipeline that can hold state.

| # | Item | Spec | Impl | Reference |
|---|---|---|---|---|
| 1.1 | Intake interview and `devrel.yaml` manifest | ✅ | ○ | [interview](../orchestrator/interview.md) · [questionnaire](../orchestrator/questionnaire.yaml) |
| 1.2 | `init` / `onboard` / `adopt` commands | ✅ | ○ | [commands](../commands/README.md) |
| 1.3 | SQLite store, schema and migrations | ✅ | ○ | [DATA_MODEL](../architecture/DATA_MODEL.md) · [ADR-0016](../adr/ADR-0016-local-first-single-file-store.md) |
| 1.4 | In-process event bus, persisted | ✅ | ○ | [EVENTS](../architecture/EVENTS.md) |
| 1.5 | Post state machine and transition ledger | ✅ | ○ | [STATE_MACHINE](../architecture/STATE_MACHINE.md) |
| 1.6 | Channel and voice profile loaders | ✅ | ○ | [channels](../orchestrator/channels/_schema.md) · [voices](../orchestrator/voices/_schema.md) |
| 1.7 | Provider abstraction with tier routing | ◐ | ○ | [ADR-0005](../adr/ADR-0005-provider-abstraction.md) · [ADR-0013](../adr/ADR-0013-cost-control-and-model-routing.md) |
| 1.8 | `config/` overlay — `defaults.yaml`, `house-rules.md`, `scoring.yaml` | ○ | ○ | — |

## M2 — Wave 0 positioning

**Goal:** the audit that gates distribution on the project being ready for it.

| # | Item | Spec | Impl |
|---|---|---|---|
| 2.1 | Deterministic repo checks — demo above the fold, install command, topics, description, licence | ✅ | ○ |
| 2.2 | Rubric-scored positioning review against `positioning.comparables` | ✅ | ○ |
| 2.3 | `launch` writes `positioning.last_audit`; campaigns gate on its score | ✅ | ○ |

## M3 — Mining and drafting

**Goal:** repository activity becomes ranked candidates, then drafts.

| # | Item | Spec | Impl | Reference |
|---|---|---|---|---|
| 3.1 | Deterministic scorer, weights in `config/scoring.yaml` | ✅ | ○ | [mine](../commands/mine.md) |
| 3.2 | Hard exclusions and dedup against the past-post index | ✅ | ○ | [mine](../commands/mine.md) |
| 3.3 | Angle, Copywriter, Reviewer nodes with the iteration cap | ✅ | ○ | [agents](../agents/README.md) · [draft](../commands/draft.md) |
| 3.4 | Voice fingerprint application and calibration loop | ✅ | ○ | [ADR-0012](../adr/ADR-0012-voice-profile-and-calibration.md) |
| 3.5 | Knowledge base — FTS5 + embeddings | ◐ | ○ | [ADR-0016](../adr/ADR-0016-local-first-single-file-store.md) |

## M4 — Gate and publish, `auto` tier

**Goal:** the first post reaches a channel, through a human.

| # | Item | Spec | Impl | Reference |
|---|---|---|---|---|
| 4.1 | `PrePublishGate` — both passes, all eight stages | ✅ | ○ | [ADR-0014](../adr/ADR-0014-deterministic-pre-publish-gate.md) |
| 4.2 | Review queue, `approve` / `reject`, `edit-required` enforcement | ✅ | ○ | [DATA_MODEL](../architecture/DATA_MODEL.md) |
| 4.3 | Outbox, idempotency keys, reconciliation | ✅ | ○ | [publish](../commands/publish.md) |
| 4.4 | Dev.to adapter (`auto` tier) | ✅ | ○ | [devto](../orchestrator/channels/devto.yaml) |
| 4.5 | `pause` / `resume` kill switch, with re-gating on resume | ✅ | ○ | [pause](../commands/pause.md) |
| 4.6 | `retract` runbook, branching on `api.delete` | ✅ | ○ | [retract](../commands/retract.md) |

## M5 — `assisted` and `draft` tiers

**Goal:** the channels that cannot be fully automated, handled honestly.

| # | Item | Spec | Impl | Reference |
|---|---|---|---|---|
| 5.1 | Quota metering and refusal on exhaustion | ✅ | ○ | [ADR-0011](../adr/ADR-0011-channel-capability-tiers.md) |
| 5.2 | LinkedIn adapter, gated on app approval | ✅ | ○ | [linkedin](../orchestrator/channels/linkedin.yaml) |
| 5.3 | `draft`-tier hand-off — composer link, checklist, URL return | ✅ | ○ | [hackernews](../orchestrator/channels/hackernews.yaml) |
| 5.4 | Further profiles — Hashnode, Mastodon, Discord, GitHub Releases, X, Reddit | ○ | ○ | — |
| 5.5 | GitHub-native surface — topics, description, README badges | ○ | ○ | — |

## M6 — Measurement

**Goal:** know what happened, without claiming more than the data supports.

| # | Item | Spec | Impl | Reference |
|---|---|---|---|---|
| 6.1 | Daily append-only snapshot, gaps recorded as gaps | ✅ | ○ | [ADR-0015](../adr/ADR-0015-attribution-methodology.md) |
| 6.2 | Owned redirect and UTM tagging | ✅ | ○ | [ADR-0015](../adr/ADR-0015-attribution-methodology.md) |
| 6.3 | Pre/post lift against a control window, reported as directional | ✅ | ○ | [ADR-0015](../adr/ADR-0015-attribution-methodology.md) |
| 6.4 | `digest` and `audit` — spend per stage, gate false-positive rate, human-gate decay | ✅ | ○ | [digest](../commands/digest.md) · [audit](../commands/audit.md) |

## M7 — Verification harness

**Goal:** the claims above become checkable.

| # | Item | Spec | Impl | Reference |
|---|---|---|---|---|
| 7.1 | `eval` runner — per-class scoring, variance, baselines | ✅ | ○ | [eval](../commands/eval.md) |
| 7.2 | Miner golden set with labelling protocol | ✅ | ○ | [eval/miner](../eval/miner.md) |
| 7.3 | Reviewer corpus, eight defect classes | ✅ | ○ | [eval/reviewer](../eval/reviewer.md) |
| 7.4 | Adversarial corpus — injection containment, leak recall | ✅ | ○ | [eval/adversarial](../eval/adversarial.md) |
| 7.5 | Channel contracts and formatter snapshots | ✅ | ○ | [eval/channels](../eval/channels.md) |
| 7.6 | Cost regression suite | ✅ | ○ | [eval/cost](../eval/cost.md) |
| 7.7 | State machine property tests | ✅ | ○ | [STATE_MACHINE](../architecture/STATE_MACHINE.md) |

---

## Beyond V1 — not specified, deliberately

These are named so the shape of the product is clear, not committed to. None has an ADR, and none
should acquire one before V1 ships.

- **Web dashboard** for the review queue and metrics.
- **Multi-repository workspace.** A genuinely different deployment with a different store
  ([ADR-0016](../adr/ADR-0016-local-first-single-file-store.md)); the `repo_id` column exists so it
  is a data move rather than a redesign.
- **Automated terminal-demo capture** — the 60–90 s GIF Wave 0 asks for.
- **Awesome-list submission assistance**, at `draft` tier: quality pre-checks and a prepared PR, with
  a human submitting. Automating list submissions is the same class of mistake as automating Show HN.
- **A public plugin API.** [ADR-0004](../adr/ADR-0004-plugin-system.md) commits to one before three
  adapters exist. The honest sequence is an internal adapter interface now, a public API when
  third-party demand is real — and a plugin sandbox before any third-party code runs next to
  publishing credentials.
