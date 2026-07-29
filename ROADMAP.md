# Roadmap — pgs-eadros

The project's plan as a numbered, checkbox-driven list. When an item completes in a PR,
flip its checkbox (`- [ ]` → `- [x]`) **in the same PR**. New work goes at the bottom of
its section with a fresh `<milestone>.<task>` number; never renumber.

- **Versioning start:** pre-1.0 milestone-driven.
- **Session journal:** see [`.eadros-core/docs/journal/`](.eadros-core/docs/journal/). Latest checkpoint: _none yet_.

## Model & effort routing (advisory)

An item may carry an advisory **route** — `route: <tier> / <effort>` — derived from its intake
signals through the `os/routing` policy's only-raise resolution (ADR-0017: start at the floor;
matched signals only ever raise, never lower). Tiers, cheapest → most capable: fast → standard → frontier-reasoning.
Efforts: low → medium → high → extra → max. An item with no route takes the floor (fast / low). The route
*recommends*; **the human keeps final model authority** — switch with your host's own model
control, never mid-session by the agent.

Tiers map to concrete models only through the dated catalog (as of 2026-07-27;
a stale date is the review cue):

- **claude-code**: fast → Sonnet 5 · standard → Opus 5 · frontier-reasoning → Fable 5
- **codex**: fast → GPT Luna · standard → GPT Terra · frontier-reasoning → GPT Sol
- **gemini**: fast → — · standard → — · frontier-reasoning → —
- **opencode**: fast → Sonnet 5 · standard → Opus 5 · frontier-reasoning → Fable 5

Where the EADOS core is vendored (`.eados-core/`), the authoritative per-issue call once tracker
labels exist is `python .eados-core/tools/route_advice.py --issue <N>`.

---

## Milestone 1 — Project bootstrap & CI

The thinnest slice that compiles, tests, and ships under the full quality bar.

- [ ] 1.1 Lay down the build system (tsup (esbuild) / tsc --build) and a buildable skeleton under
      `.eadros-core/src/main/typescript/dev/d4np/eadros/`.
- [ ] 1.2 Wire the test framework (Vitest (or Jest)) with one passing smoke test under
      `.eadros-core/src/test/typescript/dev/d4np/eadros/`.
- [ ] 1.3 Add formatter + linter configs (Prettier, ESLint (typescript-eslint, type-aware) + tsc --noEmit --strict) at the repo root.
- [ ] 1.4 Stand up the CI matrix (Linux / Windows / macOS on Node 22, 24 LTS) with build + test + format + lint.
- [ ] 1.5 Seed the version constant (export const VERSION = 'X.Y.Z') in `version.ts`.
- [ ] 1.6 ADR recording D6 - .eadros-core/ as the installation root - and the concrete internal layout derived from it (RFC-0001 D6, closes O5) — route: frontier-reasoning / extra (adr)
- [ ] 1.7 Verify the Node support dates underpinning D7 against the published release schedule (RFC-0001 D7)
- [ ] 1.8 Run the FTS5 probe for node:sqlite on Node 22 and 24 and record the result against the D8 migration trigger (RFC-0001 D8)
- [ ] 1.9 Prebuild and cold-start gate - per matrix cell on a clean image, assert install resolves a prebuilt better-sqlite3 and never invokes a compiler; measure cold npx against the 60 s budget (RFC-0001 D8) — route: standard / high (severity:high)
- [ ] 1.10 Non-invasive-install gate (ADR-0004) - install into a POPULATED fixture repo on a clean image and fail on any path outside the closed allowlist (.eadros-core/**, .claude/commands/eadros/**, a consented one-line .gitignore append) AND on any checksum change to a pre-existing file — route: frontier-reasoning / extra (security)
- [ ] 1.11 Clean-uninstall assertion (ADR-0004) - rm -rf .eadros-core/ plus the two allowlisted paths returns the fixture repo to its pre-install checksum set — route: standard / high (severity:high)


---

## Milestone 2 — Foundation

A manifest, a store, and a pipeline that can hold state.

- [ ] 2.1 Intake interview and devrel.yaml manifest (spec M1.1) — route: standard / medium (severity:medium)
- [ ] 2.2 init / onboard / adopt commands (spec M1.2) — route: standard / medium (severity:medium)
- [ ] 2.3 SQLite store, schema and forward-only numbered migrations per RFC-0001 O3 (spec M1.3) — route: standard / high (severity:high)
- [ ] 2.4 In-process event bus, persisted in the same transaction as the state change (spec M1.4) — route: standard / high (severity:high)
- [ ] 2.5 Post state machine and transition ledger (spec M1.5) — route: standard / high (severity:high)
- [ ] 2.6 Channel and voice profile loaders, including the 90-day staleness refusal (spec M1.6) — route: standard / medium (severity:medium)
- [ ] 2.7 Provider abstraction with tier routing (spec M1.7 - partial in the spec) — route: frontier-reasoning / high (sets-pattern)
- [ ] 2.8 config/ overlay - defaults.yaml, house-rules.md, scoring.yaml (spec M1.8 - not yet specified)
- [ ] 2.9 Store-module interface isolating the SQLite driver, so the node:sqlite swap is one file (RFC-0001 D8) — route: frontier-reasoning / high (sets-pattern)
- [ ] 2.10 Error taxonomy and CLI exit-code mapping for the eleven refusal classes (RFC-0001 O4) — route: standard / medium (severity:medium)
- [ ] 2.11 Host command-adapter generation - the /eadros <name> surface generated per host from the canonical procedures, never hand-written (commands/README.md two-layer convention) — route: frontier-reasoning / high (sets-pattern)
- [ ] 2.12 Per-channel target identity - which account or page a channel publishes to, with the tier resolved per target rather than per platform (LinkedIn personal profile is draft, approved company app is assisted) — route: frontier-reasoning / extra (security)


---

## Milestone 3 — Wave 0 positioning

The audit that gates distribution on the project being ready for it.

- [ ] 3.1 Deterministic repo checks - demo above the fold, install command, topics, description, licence (spec M2.1) — route: standard / medium (severity:medium)
- [ ] 3.2 Rubric-scored positioning review against positioning.comparables (spec M2.2) — route: standard / medium (severity:medium)
- [ ] 3.3 launch writes positioning.last_audit; campaigns gate on its score (spec M2.3) — route: standard / medium (severity:medium)


---

## Milestone 4 — Mining and drafting

Repository activity becomes ranked candidates, then drafts.

- [ ] 4.1 Deterministic scorer, weights in config/scoring.yaml - identical input yields identical output (spec M3.1) — route: standard / high (severity:high)
- [ ] 4.2 Hard exclusions and dedup against the past-post index (spec M3.2) — route: standard / high (severity:high)
- [ ] 4.3 Angle, Copywriter and Reviewer nodes with the hard iteration cap (spec M3.3) — route: frontier-reasoning / high (sets-pattern)
- [ ] 4.4 Voice fingerprint application and the human calibration loop (spec M3.4) — route: standard / high (severity:high)
- [ ] 4.5 Knowledge base - FTS5 plus an exhaustive cosine scan over the embeddings table (spec M3.5 - partial) — route: standard / high (severity:high)


---

## Milestone 5 — Gate and publish, auto tier

The first post reaches a channel, through a human.

- [ ] 5.1 PrePublishGate - both passes, all eight stages; secrets and taint with no off switch (spec M4.1) — route: frontier-reasoning / extra (security)
- [ ] 5.2 Review queue, approve / reject, edit-required enforcement (spec M4.2) — route: standard / high (severity:high)
- [ ] 5.3 Outbox, idempotency keys, reconciliation that asks the platform and never retries blind (spec M4.3) — route: standard / high (severity:high)
- [ ] 5.4 Dev.to adapter, auto tier - the first adapter fixes the template every later one copies (spec M4.4) — route: frontier-reasoning / high (sets-pattern)
- [ ] 5.5 pause / resume kill switch, with the held queue re-gated on resume (spec M4.5) — route: frontier-reasoning / extra (security)
- [ ] 5.6 retract runbook, branching on the channel api.delete capability (spec M4.6) — route: standard / high (severity:high)


---

## Milestone 6 — assisted and draft tiers

The channels that cannot be fully automated, handled honestly.

- [ ] 6.1 Quota metering and refusal on exhaustion (spec M5.1) — route: standard / medium (severity:medium)
- [ ] 6.2 LinkedIn adapter, gated on app approval (spec M5.2) — route: standard / medium (severity:medium)
- [ ] 6.3 draft-tier hand-off - composer link, checklist, URL return, and no dispatch path in code (spec M5.3) — route: frontier-reasoning / extra (security)
- [ ] 6.4 Further channel profiles - Hashnode, Mastodon, Discord, GitHub Releases, X, Reddit (spec M5.4 - not yet specified) — route: standard / medium (severity:medium)
- [ ] 6.5 GitHub-native surface - PROPOSE topics, description and README badges as a printed diff the maintainer applies; never write a file the host project owns (ADR-0004; spec M5.5 - not yet specified) — route: standard / medium (severity:medium)


---

## Milestone 7 — Measurement

Know what happened, without claiming more than the data supports.

- [ ] 7.1 Daily append-only snapshot, gaps recorded as gaps - a missed day is lost permanently (spec M6.1) — route: standard / high (severity:high)
- [ ] 7.2 Owned redirect and UTM tagging (spec M6.2) — route: standard / medium (severity:medium)
- [ ] 7.3 Pre/post lift against a control window, reported as directional (spec M6.3) — route: standard / medium (severity:medium)
- [ ] 7.4 digest and audit - spend per stage, gate false-positive rate, human-gate decay (spec M6.4) — route: standard / medium (severity:medium)


---

## Milestone 8 — Verification harness

The claims above become checkable.

- [ ] 8.1 eval runner - per-class scoring, variance, baselines (spec M7.1) — route: standard / high (severity:high)
- [ ] 8.2 Miner golden set with its labelling protocol and inter-rater agreement (spec M7.2) — route: standard / high (severity:high)
- [ ] 8.3 Reviewer corpus, eight defect classes, gated per class (spec M7.3) — route: standard / high (severity:high)
- [ ] 8.4 Adversarial corpus - injection containment and leak recall, both hard-gated at 1.0 (spec M7.4) — route: frontier-reasoning / extra (security)
- [ ] 8.5 Channel contracts and formatter snapshots (spec M7.5) — route: standard / medium (severity:medium)
- [ ] 8.6 Cost regression suite (spec M7.6) — route: standard / medium (severity:medium)
- [ ] 8.7 State machine property tests - the six invariants the schema cannot express (spec M7.7) — route: standard / high (severity:high)


---

## Milestone 9 — Release and distribution

The tool is actually installable by the one command its install-cost requirement names.

- [ ] 9.1 Dual ESM/CJS package with an explicit export map (RFC-0001 D7) — route: standard / medium (severity:medium)
- [ ] 9.2 version.ts and package.json version lockstep, asserted by the consistency lint (RFC-0001 D7)
- [ ] 9.3 Release workflow - tag, changelog, and the agent-drafts / human-publishes boundary (RFC-0001) — route: standard / medium (severity:medium)
- [ ] 9.4 Published-artifact acceptance - install the packed tarball on a clean image per matrix cell and re-run the 1.9 prebuild and cold-start assertions against the real package (RFC-0001 D8) — route: standard / high (severity:high)



---

## Spec Coverage Map

Tracks which spec section is fulfilled by which roadmap item(s). Every spec section has a
row with at least one fulfilling item and a status glyph. Legend: ⏳ not started · 🚧 in
progress · ✅ done · ❎ N/A.

| Spec § | Requirement | Roadmap items | Status |
|--------|-------------|---------------|--------|
| §1 | Objective & business context | 1.1 | ⏳ |
| §2 | Functional requirements | 1.1, 1.2 | ⏳ |
| §3 | Non-functional requirements | 1.3, 1.4 | ⏳ |
| §4 | Logical architecture | 1.1 | ⏳ |
| §5 | Public interface | 1.2 | ⏳ |
| §6 | Verification & test strategy | 1.2, 1.4 | ⏳ |
