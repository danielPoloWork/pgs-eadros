# Software Specification: EADROS — Enterprise Agentic Developer Relations OS (TypeScript)

> Rendered from the intake interview (Phase 5). Frozen contract: diverging implementation
> updates this spec in the same PR or adds an ADR superseding the relevant section.

## 1. Objective & Business Context

Transform every engineering activity into discoverable, high-impact technical content
(vision/VISION.md). EADROS observes real engineering activity in a repository — commits,
pull requests, releases, ADR changes, CI gate intercepts — identifies the events that carry
a technical story, and runs a governed agent pipeline that drafts, checks and distributes
it. A human approves everything that ships.

It exists because high-quality open-source projects fail from absent distribution, not
absent quality: maintainers have no time for Developer Relations, and generic social-media
tooling produces the promotional register developer communities reject on sight.

Three constraints shape the system, and none of them is "generate good text" — text
generation is the commodity part. (1) Most destinations forbid what is being automated, and
over-automation can get the promoted project's domain permanently banned from the
highest-signal channel in its ecosystem (ADR-0011). (2) Detectably generated output
falsifies the entire thesis in one post, unrecoverably — the product is authenticity, so a
recognisable model register is a category failure, not a quality issue (ADR-0012). (3) The
pipeline reads private repositories, holds publishing credentials, and ingests text any
stranger can write on a public repository's pull request (ADR-0014).

Scope: one repository per program, a solo maintainer or small team, at most two substantive
posts per week, 6-8 channels. Local-first: one CLI, one SQLite file, no servers.

## 2. Functional Requirements

- Story mining: score repository activity deterministically; only the top K reach a model.
- Governed drafting: angle -> draft -> review, per channel and locale, within an iteration cap.
- Channel-native output: each destination's formatting rules enforced from its profile.
- Tiered publishing: `auto` dispatches on approval; `assisted` dispatches under quota; `draft` never dispatches — it prepares a payload, a composer link and a checklist for a human, then records the returned URL so the channel stays instrumented.
- Human approval gate: nothing publishes without an approval record. Approval is invalid unless the maintainer edited the draft, or passed `--as-is`, which is recorded.
- Retraction: a runbook branching on whether the channel supports removal at all.
- Measurement: daily append-only snapshots; lift reported as directional, never causal.
- Onboarding interview: channels, voice, safety and budget elicited once, with provenance recorded per answer; imports the EADOS manifest where present rather than re-asking.
- Wave 0 positioning gate: `launch` audits whether the project explains itself (value proposition in the first screen, 60-90 s demo above the fold, install command visible, high-intent keywords, repo topics and description set) and campaigns declare a pre-condition on its score.


## 3. Non-Functional Requirements

<!-- Scalability / load budgets belong here as NUMBERS, not adjectives (the design "scalability"
     fold): a value per hard NFR axis — throughput / concurrency, p99 latency, memory ceiling,
     target FPS, cold-start budget — each phrased so CI could prove a violation. -->
- Model calls during mining: exactly 0 — verified by the eval cost suite.
- Cost per published post: <= EUR 0.15 — measured per stage via `budget_ledger`.
- Mining latency: a 500-commit window scored in <= 10 s — miner benchmark.
- Candidate to review queue: p95 <= 3 min — pipeline timestamps.
- Human review time per post: <= 5 min — review queue timestamps.
- Leak-gate recall on secrets: 1.0, an absolute — adversarial suite, hard gate with no threshold negotiation.
- Prompt-injection containment: 1.0, an absolute — adversarial suite; success is containment (attacker text never reached a credentialed component as instruction), not model refusal.
- Double-publish rate: 0, an absolute — property test plus `posts.idempotency_key`.
- Gate false-positive rate: <= 0.10 — `gate_verdicts.false_positive`, reported by `audit`.
- `auto`-tier publish success: >= 99% excluding platform outages — outbox outcomes.
- Metrics snapshot gap rate: <= 1% of days — `post_metrics.is_gap`.
- Install cost: 0 services; a cold `npx` run in <= 60 s — CI on a clean image.
- Installation containment: every asset the product installs lives under `.eadros-core/`, the installation root; nothing is written outside it except the maintainer's own manifest. Mechanically checkable by asserting the post-install file set on a clean image.
- Channel profile freshness: a profile older than 90 days blocks onboarding, because a compliance rule that cannot be re-verified is silently wrong within a year.
- Observability: every event persisted with `correlation_id` and `causation_id`, so one query returns the complete account of how one CI failure became three posts.


## 4. Logical Architecture & Core Algorithm

<!-- For a non-obvious core algorithm, include a short LANGUAGE-FREE pseudocode sketch (control
     flow + invariants) alongside the prose + diagram (the design "pseudocode" fold); skip it when
     the approach is standard. If the design owns persistent state, capture the data model here —
     entities, relations, normal form, migration policy — within ADR-0004's secondary-SQL frame. -->
One process, one file, no services (architecture/CONTAINERS.md). A Node.js/TypeScript CLI
holds an in-process event bus whose events are persisted to the same SQLite file in the same
transaction as the state change they describe; the knowledge base (FTS5 + embeddings), the
outbox, the budget ledger and the metrics time series live in that one file (ADR-0016).

The orchestrator is a FIXED DAG with models inside the nodes, not an agent that plans the
pipeline. Letting a model choose the graph would add non-determinism, unbounded cost and an
unauditable trail to a system whose entire value is governance.

  repository activity
    -> 1 Story miner      (deterministic, no model call, ever)
    -> 2 PrePublishGate   (deterministic, pass 1 of 2, BEFORE any model sees the content)
    -> 3 Angle            (LLM, cheap tier)
    -> 4 Copywriter       (LLM, mid tier)  <-- critique --  5 Reviewer (LLM, strong tier)
                          (max 2 iterations, hard)
    -> 2 PrePublishGate   (deterministic, pass 2 of 2, + claims resolution, voice lint)
    -> 6 Review queue     (human gate — no agent path exists)
    -> 7 Scheduler        (deterministic: channel ceilings, window, preconditions)
    -> 8 Publisher        (deterministic: tier branch, outbox, idempotency, reconcile)
         auto | assisted | draft -> a human posts
    .. 9 Metrics collector (deterministic, daily snapshot)
  Knowledge base supports 3, 4, 5 and the miner's dedup; it is not in the DAG.

Three of the nine components hold a model — framing, writing and critique, where judgment
genuinely lives. The other six carry a GUARANTEE rather than a judgment, and a guarantee
implemented in a prompt can only ever be an aspiration: the miner must be reproducible or the
cost model has no floor; the gate must have recall 1.0 on planted secrets; the publisher must
never double-post; the scheduler must never exceed a channel ceiling.

Capability tiers (ADR-0011) are derived from each destination's dated profile and stated to
the maintainer, never offered as a choice: they choose WHETHER to use a channel, the profile
decides HOW. `draft` is not a missing feature but the correct permanent answer for platforms
whose value depends on submissions being human, and such adapters have no dispatch path in
code at all.

INSTALLATION ROOT — a binding architectural decision, confirmed by the maintainer on
2026-07-29 and to be PRESERVED through every later phase. The runtime is designed so that
every asset the product installs is contained in `.eadros-core/`, which IS the installation
root of the system. The directory is already the unit the spec treats as movable — `upgrade`
migrates `.eadros-core/` plus the manifest to a newer core version (commands/upgrade.md) —
and this decision makes that containment a property of the design rather than a convention:
one directory to install, to upgrade, to inspect and to remove. The design phase is to record
it as an ADR and derive the concrete layout from it; it is not open for silent revision.

Canonical detail (not duplicated here): architecture/SYSTEM_CONTEXT.md, CONTAINERS.md,
COMPONENTS.md, DATA_MODEL.md, EVENTS.md, STATE_MACHINE.md, SEQUENCE_DIAGRAMS.md.

## 5. Public Interface

<!-- The API contract (the design "api" fold): each operation with its payload shapes, the error
     model (the failure taxonomy, not just the happy path), and the versioning / SemVer surface.
     A service/web project may keep the written-out contract under docs/api/ (capabilities.api_spec). -->
Consumers import via `import { ... } from '@d4np/eadros';`. The public surface:

- `eadros init` — frames the program: interview Phase 0-1, imports the EADOS manifest, writes the devrel.yaml skeleton. Drafts; the human confirms.
- `eadros onboard [--recalibrate]` — the channel + voice interview, ending in the voice calibration loop. The human answers and owns every tier acknowledgement.
- `eadros adopt` — brownfield intake for a repo that already posts: read-only presence map, goal menu, `adoption:` block. Proposes, never migrates.
- `eadros doctor` — preflight: tokens present, scopes sufficient, quota remaining, credentials near expiry, channel policies older than 90 days. Reports; fixes nothing.
- `eadros status` — where every campaign sits in the state machine, budget consumed, review queue depth. Read-only.
- `eadros upgrade` — migrates .eadros-core/ and the manifest to a newer core version. Proposes a diff; the human applies.
- `eadros launch` — the Wave 0 audit: deterministic repo checks plus a rubric-scored positioning review; writes positioning.last_audit. A gate, not a report.
- `eadros mine` — deterministic story scoring, no model calls; prints scored candidates with their signals. Read-only, free, inspectable.
- `eadros draft <id>` — runs the agent pipeline for one mined candidate through to the review queue. Drafts; never publishes.
- `eadros weekly` — the cadence run: mine, rank and draft within cadence.weekly and every channel ceiling. Drafts; never publishes.
- `eadros release` — fires on release.published: mines release notes, breaking changes, trade-offs. Drafts; never publishes.
- `eadros campaign <id>` — runs a milestone campaign after evaluating its precondition; refuses when the precondition is unmet.
- `eadros review` — opens the human review queue on the configured surface.
- `eadros approve --id <id> [--as-is]` — HUMAN ONLY; the agent may never call it. Rejected under approval_mode: edit-required unless the final text differs from the draft.
- `eadros reject --id <id> --reason <text>` — human only; the reason feeds the eval corpus.
- `eadros publish` — dispatches approved content: auto publishes, assisted publishes under quota, draft never dispatches (payload, composer link and checklist instead).
- `eadros retract --id <id>` — the wrong-post runbook, branching on the channel's api.delete: supported -> delete, edit-only -> correct in place, unsupported -> draft a correction and say plainly that removal is impossible.
- `eadros pause` / `eadros resume` — the kill switch: anyone may pause, only governance.kill_switch_owner may resume, and resume re-gates the held queue.
- `eadros metrics` — the daily append-only traffic snapshot. Not optional and not on-demand: the GitHub Traffic API retains roughly 14 days.
- `eadros digest` — the maintainer summary: what shipped, what it cost, what moved. Read-only.
- `eadros audit` — governance audit: budget adherence, leak-gate false-positive rate, tier compliance, stale channel policies, approvals that used --as-is.
- `eadros eval` — runs the verification suites: miner ranking, reviewer per-class detection, injection containment, leak recall, channel contracts, cost regression.
- Global flags: `--dry-run` (full pipeline against a mock publisher; the default for a program's first week), `--explain` (scoring signals, model routed per stage, token/cost estimate before spending), `--budget-check` (refuses to start when the run would exceed budget.cost_per_week).
- Deliberately absent: there is no `eadros auto` and no daemon that publishes unattended. Every path to a live post runs through `approve`, and `approve` is a human verb.


## 6. Verification & Test Strategy

Six suites, gated PER CLASS with no aggregate score (eval/README.md). The organising decision
is that every safety guarantee sits in a deterministic component, so it can be a test result
rather than a hope.

- Deterministic suites block CI on any regression: miner ranking (exclusion safety gated at
  1.0), channel contracts and formatter snapshots, cost regression, and the six state-machine
  property tests.
- Generative suites (Angle, Copywriter, Reviewer) run N >= 20 times per case and gate on the
  LOWER CONFIDENCE BOUND, never the mean — a prompt change that improves the average while
  doubling the variance has made the system worse.
- The adversarial suite is a hard gate with no threshold negotiation: prompt-injection
  containment and leak recall on secrets must both be 1.0. Success is containment — attacker
  text never reached a credentialed component as instruction — not "the model refused", which
  is a property of a disposition rather than of a structure.
- Every component must beat a declared trivial baseline: miner vs rank-by-diff-size, reviewer
  vs a regex over the forbidden-word list, angle vs always-pick-the-most-frequent-archetype,
  copywriter vs the channel's release-note template filled. A model-based stage that cannot
  is DELETED, not tuned.
- Metrics are reported per defect class, never aggregate: a reviewer that catches 100% of
  buzzwords and 20% of subtly-false technical claims scores well in aggregate and is useless.
- The corpus is a versioned artifact with provenance per case (synthesised / observed /
  from_retraction), the labeller, the date, and inter-rater agreement for multi-labeller
  classes.
- Production failures become regression tests: every retraction writes a permanent case
  carrying the gate verdict that wrongly passed it (retractions.gate_verdict_id). That is the
  only mechanism by which the gate improves from having been wrong.
- Model calls in CI use recorded fixtures by default; a nightly job runs the generative suites
  against live providers, because a suite that only ever sees fixtures eventually measures the
  fixtures.

Deliberately NOT tested, and stated so an eval directory does not imply coverage it lacks:
whether a post lands well with developers (only directionally measurable, ADR-0015); whether
the voice is authentically the maintainer's (settled by a human calibration loop, ADR-0012);
and whether a platform's terms still permit what its profile claims (a human re-verification
on a 90-day clock, surfaced by `doctor`).

Toolchain: built with tsup (esbuild) / tsc --build, tested with Vitest (or Jest), checked with
tsc --strict (type soundness), vitest --detectOpenHandles (leak/handle), eslint --max-warnings 0, coverage target ≥ 80% line. Every functional and
non-functional requirement above maps to a CI gate (see [`.github/workflows/ci.yml`](../../../.github/workflows/ci.yml)).
