# Software Specification: EADROS — Enterprise Agentic Developer Relations OS

| | |
|---|---|
| **Version** | 2.0 (rewritten in English per the repo's v2.0 convention; v1.0 was an Italian summary of a 684-line doc set) |
| **Date** | 2026-07-29 |
| **Status** | Reviewed draft — **specification stage, no implementation** ([roadmap](docs-eadros/roadmap/MVP.md)) |
| **Detail** | [`docs-eadros/`](docs-eadros/commands/README.md) — 16 ADRs, C4 views, data model, state machine, event contract, intake interview, eval strategy |
| **Key ADRs** | [ADR-0011: Channel capability tiers](docs-eadros/adr/ADR-0011-channel-capability-tiers.md) · [ADR-0012: Voice](docs-eadros/adr/ADR-0012-voice-profile-and-calibration.md) · [ADR-0014: Pre-publish gate](docs-eadros/adr/ADR-0014-deterministic-pre-publish-gate.md) · [ADR-0015: Attribution](docs-eadros/adr/ADR-0015-attribution-methodology.md) · [ADR-0016: Local-first store](docs-eadros/adr/ADR-0016-local-first-single-file-store.md) |

## 1. Objective & Scenario

High-quality open-source projects fail from **absent distribution**, not absent quality. Maintainers
have no time for Developer Relations, and generic social-media tooling produces the promotional
register developer communities reject on sight.

**EADROS** observes real engineering activity — commits, pull requests, releases, ADR changes, CI
gate intercepts — identifies the events that carry a technical story, and runs a governed agent
pipeline that drafts, checks, and distributes it. A human approves everything that ships.

**Three constraints shape the system, and none of them is "generate good text"** — text generation is
the commodity part. The specification is built around the three that are not:

1. **Most destinations forbid what is being automated.** Hacker News has no write API and bans by
   *domain*; Reddit's self-promotion rules are enforced per subreddit; LinkedIn gates posting behind
   app review. A tool that over-automates can get the domain of the project it was hired to promote
   permanently banned from the highest-signal channel in the ecosystem
   ([ADR-0011](docs-eadros/adr/ADR-0011-channel-capability-tiers.md)).
2. **Detectably generated output falsifies the entire thesis**, in one post, unrecoverably. The
   product is authenticity; a recognisable model register is not a quality issue but a category
   failure ([ADR-0012](docs-eadros/adr/ADR-0012-voice-profile-and-calibration.md)).
3. **The pipeline reads private repositories, holds publishing credentials, and ingests text any
   stranger can write** on a public repository's pull request
   ([ADR-0014](docs-eadros/adr/ADR-0014-deterministic-pre-publish-gate.md)).

**Scope:** one repository per program, a solo maintainer or small team, **≤ 2 substantive posts per
week**, 6–8 channels. Local-first: one CLI, one SQLite file, no servers.

**Wave 0 — positioning precedes distribution.** Before any campaign fires, `launch` audits whether
the project explains itself: value proposition legible in the first screen, a working 60–90 s demo
above the fold, install command visible, high-intent keywords present, repo topics and description
set. **This is a gate, not advice** — campaigns declare a pre-condition on its score. Distributing a
project whose README does not explain itself converts the one launch you get into a bounce.

## 2. Functional Requirements

* **Story mining:** score repository activity **deterministically**; only the top K reach a model.
* **Governed drafting:** angle → draft → review, per channel and locale, within an iteration cap.
* **Channel-native output:** each destination's formatting rules enforced from its profile.
* **Tiered publishing:** `auto` dispatches on approval; `assisted` dispatches under quota; `draft`
  **never dispatches** — it prepares a payload, a composer link and a checklist for a human, then
  records the returned URL so the channel stays instrumented.
* **Human approval gate:** nothing publishes without an approval record. Approval is invalid unless
  the maintainer edited the draft, or passed `--as-is`, which is recorded.
* **Retraction:** a runbook branching on whether the channel supports removal at all.
* **Measurement:** daily append-only snapshots; lift reported as directional, never causal.
* **Onboarding interview:** channels, voice, safety and budget elicited once, with provenance
  recorded per answer; imports the EADOS manifest where present rather than re-asking.

## 3. Non-Functional Requirements (quantified — measurement stated)

| NFR | Target | Measurement |
|---|---|---|
| Model calls during mining | **exactly 0** | [`eval` cost suite](docs-eadros/eval/cost.md) |
| Cost per published post | ≤ **€0.15** | `budget_ledger`, per stage |
| Mining latency | 500-commit window scored ≤ **10 s** | miner benchmark |
| Candidate → review queue | p95 ≤ **3 min** | pipeline timestamps |
| Human review time per post | ≤ **5 min** | review queue timestamps |
| Leak-gate recall on secrets | **1.0** | [adversarial suite](docs-eadros/eval/adversarial.md) |
| Prompt-injection containment | **1.0** | [adversarial suite](docs-eadros/eval/adversarial.md) |
| Double-publish rate | **0** | property test + `posts.idempotency_key` |
| Gate false-positive rate | ≤ **0.10** | `gate_verdicts.false_positive`, via `audit` |
| `auto`-tier publish success | ≥ **99%** excluding platform outages | outbox outcomes |
| Metrics snapshot gap rate | ≤ **1%** of days | `post_metrics.is_gap` |
| Install cost | **0 services**; cold `npx` run ≤ 60 s | CI on a clean image |

Three targets are absolutes rather than trade-offs — secrets recall, injection containment, and
double-publish. Each sits in deterministic code precisely so it can be gated at 100%; a guarantee
implemented in a prompt could only ever be an aspiration.

## 4. Architecture (C4 — one process, one file)

```
 (Maintainer) ──▶ ┌─ EADROS process (Node/TS) ─────────────────────────┐
                  │ [CLI + review surface]                             │
                  │        │                                           │
                  │ [In-process event bus] ── persisted, same txn ──┐  │
                  │        ▼                                        │  │
                  │ [Orchestrator: FIXED DAG, models in the nodes]  │  │
                  │   ├ story miner ....... deterministic, no LLM   │  │
                  │   ├ angle · copywriter · reviewer .... LLM      │  │
                  │   ├ PrePublishGate .... deterministic, 2 passes │  │
                  │   └ publisher + outbox                          │  │
                  │        │                    │                   ▼  │
                  │ [KB: FTS5 + embeddings] ──▶ │ ──▶ [ one SQLite file ]
                  │ [Metrics collector, daily]  │     state·outbox·ledger
                  └─────────────────────────────┼─────metrics·events────┘
                                                ▼
                     [LLM providers]   [Channel adapters]   [GitHub API]
                  hosted · local Ollama  auto│assisted│draft   traffic
```

The orchestrator is a **fixed DAG with models inside the nodes, not an agent that plans the
pipeline**. Letting a model choose the graph would add non-determinism, unbounded cost and an
unauditable trail to a system whose entire value is governance. Two nodes hold no model at all — the
miner and the gate — and that is what makes cost bounded and safety testable.
Detail: [CONTAINERS](docs-eadros/architecture/CONTAINERS.md) ·
[STATE_MACHINE](docs-eadros/architecture/STATE_MACHINE.md).

## 5. The core mechanism — capability tiers and a gate that is code

**Tiers.** Each destination has a dated profile recording what the platform permits; the **tier is
derived from it and stated to the maintainer, never offered as a choice**. They choose *whether* to
use a channel; the profile decides *how*.

```
auto      documented write API, automation permitted   → dispatches on approval
assisted  API exists but metered / app-reviewed        → dispatches under quota, refuses when spent
draft     no lawful automation path                    → NO dispatch path exists in code
```

`draft` is not a missing feature; it is the correct permanent answer for platforms whose value
depends on submissions being human. A profile older than 90 days blocks onboarding, because a
compliance rule that cannot be re-verified is silently wrong within a year.

**The gate.** [ADR-0008](docs-eadros/adr/ADR-0008-human-review.md) claimed human review *eliminates*
risk. It does not — reviewers rubber-stamp by the third week, and attention is not a control. A
deterministic `PrePublishGate` runs **twice**: over mined inputs **before any model call**, and over
final approved text before dispatch. Eight stages; `secrets` and `taint` have no off switch. Running
on inputs is what stops injection — filtering only outputs would mean the injected instruction had
already executed with credentials in hand.

**Data model & state.** One SQLite file. A campaign and a post have separate lifecycles, because a
partial publish — live on one channel, awaiting a human on another, failed on a third — is the normal
outcome. Two constraints carry the governance claim structurally rather than procedurally:
`posts.approval_id NOT NULL`, and `CHECK (mode = 'as-is' OR hash_before <> hash_after)` — an unedited
approval cannot be recorded silently.
Detail: [DATA_MODEL](docs-eadros/architecture/DATA_MODEL.md) ·
[EVENTS](docs-eadros/architecture/EVENTS.md).

## 6. Verification & Test Strategy

Six suites, gated **per class** with no aggregate score
([`eval/`](docs-eadros/eval/README.md)):

* **Deterministic suites** — miner ranking (exclusion safety gated at 1.0), channel contracts and
  formatter snapshots, cost regression, state-machine property tests.
* **Generative suites** — reviewer detection across eight defect classes, run **N ≥ 20 times** with
  the gate on the **lower confidence bound**, never the mean.
* **Adversarial suite** — injection containment and leak recall, both hard-gated at 1.0. Success is
  *containment* — attacker text never reached a credentialed component as instruction — not "the
  model refused", which is a property of a disposition rather than of a structure.
* **Every component must beat a trivial baseline** (miner vs diff size; reviewer vs a regex). A
  model-based stage that cannot is deleted, not tuned.
* **Production failures become regression tests.** Each retraction writes a permanent case carrying
  the gate verdict that wrongly passed it — the only mechanism by which the gate improves from having
  been wrong.

Deliberately **not** tested: whether a post lands well (only directionally measurable, §3), whether
the voice is authentically the maintainer's (settled by a human calibration loop), and whether a
platform's terms still permit what its profile claims (a human re-verification on a 90-day clock).

## 7. Threat Model

| Threat | Control |
|---|---|
| **Prompt injection** via commit messages, PR/issue text, branch names, quoted file contents | `taint: untrusted` on ingestion, propagated to derived events; untrusted spans passed as delimited data, never as instruction; containment asserted at the boundary |
| **Confidential leak** from a private repo to a public channel | Deterministic gate: unconditional secret scanning, deny-terms, path allowlist (private repos start deny-all), diff-line cap, embargo |
| **Self-inflicted platform ban** | Tier model; `draft` channels have no dispatch path in code; upward override requires a recorded ADR quoting `ban_scope` |
| **Credential theft / over-broad scopes** | Manifest stores credential *locations*, never values; minimum scopes; expiry surfaced by `doctor` before it fails mid-campaign |
| **Double publish** | Outbox commits intent before the call; `UNIQUE(idempotency_key)`; timeouts reconcile, never retry blind |
| **Supply chain via community plugins** | No public plugin API before V1; third-party code must be sandboxed before running beside publishing credentials |
| **Reputational damage from a wrong post** | `retract` runbook; kill switch; the paused queue is re-gated under updated rules before resuming |
| **Unbounded spend** | Deterministic pre-filter; capability-tier routing; iteration cap; weekly ceiling with `pause` before it is crossed |

**Observability:** structured logs and metrics per pipeline stage; every event persisted with
`correlation_id` and `causation_id`, so `SELECT * FROM events WHERE correlation_id = ?` returns the
complete account of how one CI failure became three posts — who approved it, what the gate checked,
what it cost, what it earned.

## 8. Decision Log

[ADR-0001 Purpose](docs-eadros/adr/ADR-0001-project-purpose.md) ·
[0002 Event-driven](docs-eadros/adr/ADR-0002-event-driven-architecture.md) ·
[0003 Multi-agent](docs-eadros/adr/ADR-0003-agent-orchestration.md) ·
[0004 Plugins](docs-eadros/adr/ADR-0004-plugin-system.md) ·
[0005 Providers](docs-eadros/adr/ADR-0005-provider-abstraction.md) ·
[0006 Pipeline](docs-eadros/adr/ADR-0006-content-generation-pipeline.md) ·
[0007 Channel adapters](docs-eadros/adr/ADR-0007-social-network-adapters.md) ·
[0008 Human review](docs-eadros/adr/ADR-0008-human-review.md) ·
[0009 Metrics](docs-eadros/adr/ADR-0009-metrics-engine.md) ·
[0010 Knowledge base](docs-eadros/adr/ADR-0010-knowledge-base.md)

**Decisions that shape the system:**
* [ADR-0011 — Capability tiers: the platform decides the automation level, not the maintainer](docs-eadros/adr/ADR-0011-channel-capability-tiers.md)
* [ADR-0012 — Voice elicited by sample and enforced by lint, not described by adjectives](docs-eadros/adr/ADR-0012-voice-profile-and-calibration.md)
* [ADR-0013 — Cost: deterministic pre-filter, capability-tier routing, hard ceilings](docs-eadros/adr/ADR-0013-cost-control-and-model-routing.md)
* [ADR-0014 — A deterministic pre-publish gate, because human review decays](docs-eadros/adr/ADR-0014-deterministic-pre-publish-gate.md)
* [ADR-0015 — Attribution is directional; no causal claim the data cannot support](docs-eadros/adr/ADR-0015-attribution-methodology.md)
* [ADR-0016 — One SQLite file is the whole store; no server, no sidecar](docs-eadros/adr/ADR-0016-local-first-single-file-store.md)
