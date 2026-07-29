# EADROS Roadmap

Negotiated roadmap for **EADROS — Enterprise Agentic Developer Relations OS**, produced by the
`plan` phase from the approved [RFC-0001](docs/rfc/0001-eadros-devrel-os.md). Owned by the
**producer**; the manifest is [orchestrator/project.yaml](orchestrator/project.yaml).

**Status: specification stage — no implementation exists.** Every box below is unchecked, and that
is the accurate state.

## How to read this

- **Milestones 2–8** are the specification's own roadmap
  ([MVP.md](.specs/docs-eadros/roadmap/MVP.md)), shifted by **+1** because EADOS reserves Milestone 1
  for the universal bootstrap. Each item carries its source number — `(spec M3.1)` — so the mapping
  is reversible by inspection.
- **Milestones 1 and 9 are new**, added by this phase. They exist because RFC-0001 and the manifest's
  non-functional requirements create work the imported roadmap never listed: the installation-root
  ADR, the two open verifications, the native-addon prebuild gate, and the distribution path the
  *cold `npx` ≤ 60 s* requirement implies. Adding them here rather than discovering them at scaffold
  time is the point of the phase.
- **`size`** is a T-shirt estimate from `plan.yaml` (`XS|S|M|L|XL`) — macroscopic, not story points.
- **`route`** is the advisory model tier + effort from the [`os/routing`](.eados-core/orchestrator/os/routing/_schema.md)
  policy, **computed** with `route_advice.py`, never hand-picked. Tiers, not model names: a name
  would rot, and the dated catalog owns them. The route is advice — **model authority stays with the
  human** (ADR-0017). The signal that earned a non-floor route is named in brackets.

| Signal | Route it earns | Why (from `routing.yaml`) |
|---|---|---|
| `security` | frontier-reasoning / extra · **protected** | on security posture the cost of a subtle miss dwarfs the routing saving |
| `adr` | frontier-reasoning / extra · **protected** | an ADR-bearing item is decision-heavy by definition |
| `sets-pattern` | frontier-reasoning / high | the first of a class fixes the template every follower copies |
| `severity:high` | standard / high | core-guarantee work needs sustained effort |
| `severity:medium` | standard / medium | a significant gap deserves substantive work |
| *(none)* | fast / low | the floor — everything above it is earned |

---

## Milestone 1 — Bootstrap and installation root

**Goal:** a repository that builds, lints and tests on the RFC-0001 D7 matrix, with the installation
root fixed before any code is placed inside it.

Implements RFC-0001 **D6** (installation root), **D7** (runtime floor and CI matrix) and **D8**
(SQLite driver, prebuild gate), and closes the two verifications RFC-0001 declined to perform.

- [ ] 1.1 Build system and package manifest — `package.json`, `tsconfig.json` in strict mode, tsup/tsc build (RFC-0001 D7) — size: S · route: fast / low
- [ ] 1.2 Repository skeleton under `src/main/typescript/dev/d4np/eadros`, mirrored under `src/test/…` (RFC-0001) — size: XS · route: fast / low
- [ ] 1.3 CI on the D7 matrix — {ubuntu, windows, macos} × {Node 22, 24}, with format, lint and test jobs (RFC-0001 D7) — size: M · route: standard / medium [severity:medium]
- [ ] 1.4 ADR recording D6 — `.eadros-core/` as the installation root — and the concrete internal layout derived from it (RFC-0001 D6, closes O5) — size: S · route: frontier-reasoning / extra [adr]
- [ ] 1.5 Verify the Node support dates underpinning D7 against the published release schedule; correct D7 if they differ (RFC-0001 D7 provenance warning) — size: XS · route: fast / low
- [ ] 1.6 Run the FTS5 probe for `node:sqlite` on Node 22 and 24 and record the result against the D8 migration trigger (RFC-0001 D8) — size: XS · route: fast / low
- [ ] 1.7 Prebuild and cold-start gate — assert on a clean image, **per matrix cell**, that install resolves a prebuilt `better-sqlite3` and never invokes a compiler, and measure cold `npx` against the 60 s budget. A cell without a prebuild is a release blocker (RFC-0001 D8) — size: M · route: standard / high [severity:high]
- [ ] 1.8 Post-install containment assertion — every installed asset under `.eadros-core/`, verified as a file-set assertion on a clean image (RFC-0001 D6) — size: S · route: standard / medium [severity:medium]

## Milestone 2 — Foundation

**Goal:** a manifest, a store, and a pipeline that can hold state.

Implements RFC-0001 **D4** (one process, one SQLite file) and **D5** (governance enforced by the
schema), and closes RFC-0001 **O3** (migration policy) and **O4** (exit-code taxonomy).

- [ ] 2.1 Intake interview and `devrel.yaml` manifest (spec M1.1) — size: M · route: standard / medium [severity:medium]
- [ ] 2.2 `init` / `onboard` / `adopt` commands (spec M1.2) — size: M · route: standard / medium [severity:medium]
- [ ] 2.3 SQLite store, schema and forward-only numbered migrations per RFC-0001 O3 (spec M1.3) — size: L · route: standard / high [severity:high]
- [ ] 2.4 In-process event bus, persisted in the same transaction as the state change (spec M1.4) — size: L · route: standard / high [severity:high]
- [ ] 2.5 Post state machine and transition ledger (spec M1.5) — size: L · route: standard / high [severity:high]
- [ ] 2.6 Channel and voice profile loaders, including the 90-day staleness refusal (spec M1.6) — size: M · route: standard / medium [severity:medium]
- [ ] 2.7 Provider abstraction with tier routing (spec M1.7 — partial in the spec) — size: M · route: frontier-reasoning / high [sets-pattern]
- [ ] 2.8 `config/` overlay — `defaults.yaml`, `house-rules.md`, `scoring.yaml` (spec M1.8 — not yet specified) — size: S · route: fast / low
- [ ] 2.9 Store-module interface isolating the SQLite driver, so the `node:sqlite` swap is one file rather than a refactor (RFC-0001 D8) — size: S · route: frontier-reasoning / high [sets-pattern]
- [ ] 2.10 Error taxonomy and CLI exit-code mapping for the eleven refusal classes (RFC-0001 O4) — size: M · route: standard / medium [severity:medium]
- [ ] 2.11 Host command-adapter generation — the `/eadros <name>` surface generated per host from the canonical procedures, never hand-written (`commands/README.md` two-layer convention; RFC-0001) — size: M · route: frontier-reasoning / high [sets-pattern]
- [ ] 2.12 Per-channel **target identity** — which account or page a channel publishes to, with the tier resolved per target rather than per platform. A LinkedIn personal profile is `draft` and an approved company app is `assisted`: the same platform, two tiers, and nothing currently disambiguates them (RFC-0001 D2) — size: M · route: frontier-reasoning / extra [security]

## Milestone 3 — Wave 0 positioning

**Goal:** the audit that gates distribution on the project being ready for it.

Implements the Wave 0 functional requirement carried into RFC-0001 — `launch` is a gate, not a
report.

- [ ] 3.1 Deterministic repo checks — demo above the fold, install command, topics, description, licence (spec M2.1) — size: M · route: standard / medium [severity:medium]
- [ ] 3.2 Rubric-scored positioning review against `positioning.comparables` (spec M2.2) — size: M · route: standard / medium [severity:medium]
- [ ] 3.3 `launch` writes `positioning.last_audit`; campaigns gate on its score (spec M2.3) — size: S · route: standard / medium [severity:medium]

## Milestone 4 — Mining and drafting

**Goal:** repository activity becomes ranked candidates, then drafts.

Implements RFC-0001 **D1** (fixed DAG, models in the nodes) and the miner algorithm sketched in its
pseudocode fold.

- [ ] 4.1 Deterministic scorer, weights in `config/scoring.yaml` — identical input yields identical output, or the cost model has no floor (spec M3.1) — size: L · route: standard / high [severity:high]
- [ ] 4.2 Hard exclusions and dedup against the past-post index (spec M3.2) — size: M · route: standard / high [severity:high]
- [ ] 4.3 Angle, Copywriter and Reviewer nodes with the hard iteration cap (spec M3.3) — size: L · route: frontier-reasoning / high [sets-pattern]
- [ ] 4.4 Voice fingerprint application and the human calibration loop (spec M3.4) — size: L · route: standard / high [severity:high]
- [ ] 4.5 Knowledge base — FTS5 plus an exhaustive cosine scan over the embeddings table (spec M3.5 — partial in the spec) — size: L · route: standard / high [severity:high]

## Milestone 5 — Gate and publish, `auto` tier

**Goal:** the first post reaches a channel, through a human.

Implements RFC-0001 **D3** (the deterministic gate that runs twice) and the outbox half of **D5**.
This is the milestone where the product's safety claims stop being design and become test results.

- [ ] 5.1 `PrePublishGate` — both passes, all eight stages; `secrets` and `taint` with no off switch (spec M4.1) — size: XL · route: frontier-reasoning / extra [security]
- [ ] 5.2 Review queue, `approve` / `reject`, `edit-required` enforcement (spec M4.2) — size: L · route: standard / high [severity:high]
- [ ] 5.3 Outbox, idempotency keys, reconciliation that asks the platform and never retries blind (spec M4.3) — size: L · route: standard / high [severity:high]
- [ ] 5.4 Dev.to adapter, `auto` tier — the first adapter fixes the template every later one copies (spec M4.4) — size: M · route: frontier-reasoning / high [sets-pattern]
- [ ] 5.5 `pause` / `resume` kill switch, with the held queue re-gated on resume (spec M4.5) — size: M · route: frontier-reasoning / extra [security]
- [ ] 5.6 `retract` runbook, branching on the channel's `api.delete` (spec M4.6) — size: M · route: standard / high [severity:high]

## Milestone 6 — `assisted` and `draft` tiers

**Goal:** the channels that cannot be fully automated, handled honestly.

Implements RFC-0001 **D2** (the tier is derived from the platform's dated profile, never chosen by
the maintainer).

- [ ] 6.1 Quota metering and refusal on exhaustion (spec M5.1) — size: M · route: standard / medium [severity:medium]
- [ ] 6.2 LinkedIn adapter, gated on app approval (spec M5.2) — size: M · route: standard / medium [severity:medium]
- [ ] 6.3 `draft`-tier hand-off — composer link, checklist, URL return, and **no dispatch path in code**; the control against a self-inflicted platform ban (spec M5.3) — size: M · route: frontier-reasoning / extra [security]
- [ ] 6.4 Further channel profiles — Hashnode, Mastodon, Discord, GitHub Releases, X, Reddit (spec M5.4 — not yet specified) — size: L · route: standard / medium [severity:medium]
- [ ] 6.5 GitHub-native surface — topics, description, README badges (spec M5.5 — not yet specified) — size: S · route: fast / low

## Milestone 7 — Measurement

**Goal:** know what happened, without claiming more than the data supports.

Implements the attribution discipline RFC-0001 carries from ADR-0015: directional, never causal.

- [ ] 7.1 Daily append-only snapshot, gaps recorded as gaps — a missed day is lost permanently, not late (spec M6.1) — size: M · route: standard / high [severity:high]
- [ ] 7.2 Owned redirect and UTM tagging (spec M6.2) — size: M · route: standard / medium [severity:medium]
- [ ] 7.3 Pre/post lift against a control window, reported as directional (spec M6.3) — size: M · route: standard / medium [severity:medium]
- [ ] 7.4 `digest` and `audit` — spend per stage, gate false-positive rate, human-gate decay (spec M6.4) — size: M · route: standard / medium [severity:medium]

## Milestone 8 — Verification harness

**Goal:** the claims above become checkable.

Implements the verification strategy RFC-0001 states: per-class gating, no aggregate score, and the
lower confidence bound rather than the mean.

- [ ] 8.1 `eval` runner — per-class scoring, variance, baselines (spec M7.1) — size: L · route: standard / high [severity:high]
- [ ] 8.2 Miner golden set with its labelling protocol and inter-rater agreement (spec M7.2) — size: L · route: standard / high [severity:high]
- [ ] 8.3 Reviewer corpus, eight defect classes, gated per class (spec M7.3) — size: L · route: standard / high [severity:high]
- [ ] 8.4 Adversarial corpus — injection containment and leak recall, both hard-gated at 1.0 (spec M7.4) — size: XL · route: frontier-reasoning / extra [security]
- [ ] 8.5 Channel contracts and formatter snapshots (spec M7.5) — size: M · route: standard / medium [severity:medium]
- [ ] 8.6 Cost regression suite (spec M7.6) — size: M · route: standard / medium [severity:medium]
- [ ] 8.7 State machine property tests — the six invariants the schema cannot express (spec M7.7) — size: M · route: standard / high [severity:high]

## Milestone 9 — Release and distribution

**Goal:** the tool is actually installable by the one command its install-cost requirement names.

New in this phase. RFC-0001 and the manifest assert *0 services and a cold `npx` run ≤ 60 s* and set
`capabilities.packaging: true`, but the imported roadmap has no milestone that ships the package —
an install-cost budget with no distribution path is a claim about a thing that does not exist.

- [ ] 9.1 Dual ESM/CJS package with an explicit export map (RFC-0001 D7; the TypeScript profile's publishing note) — size: M · route: standard / medium [severity:medium]
- [ ] 9.2 `version.ts` ↔ `package.json` version lockstep, asserted by the consistency lint (RFC-0001 D7) — size: S · route: fast / low
- [ ] 9.3 Release workflow — tag, changelog, and the agent-drafts / human-publishes boundary (RFC-0001) — size: M · route: standard / medium [severity:medium]
- [ ] 9.4 Published-artifact acceptance — install the packed tarball on a clean image per matrix cell and re-run the 1.7 prebuild and cold-start assertions against the **real** package, not the working tree (RFC-0001 D8) — size: M · route: standard / high [severity:high]

---

## Not committed, deliberately

Named so the shape of the product is clear, not planned. None has an ADR, and none should acquire
one before V1 ships ([MVP.md](.specs/docs-eadros/roadmap/MVP.md) "Beyond V1"): a web dashboard for
the review queue, a multi-repository workspace, automated terminal-demo capture, awesome-list
submission assistance at `draft` tier, and a public plugin API.

RFC-0001 A9 rejects the web dashboard for V1 explicitly; A7 rejects the plugin API until three
adapters exist and a sandbox precedes any third-party code running beside publishing credentials.

## Open items carried from RFC-0001

`O3` (migration policy) and `O4` (exit codes) are scheduled — items 2.3 and 2.10. `O5` is item 1.4.
`O1` and `O2` were settled as D7 and D8, with their two residual verifications scheduled as 1.5 and
1.6. `O6` — the roadmap items the specification itself marks `○`/`◐` — is carried in place: items
2.7, 2.8, 4.5, 6.4 and 6.5 name their partial status inline rather than hiding it.
