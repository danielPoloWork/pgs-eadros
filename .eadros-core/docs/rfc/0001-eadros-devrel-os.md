# RFC-0001: EADROS — a governed pipeline from engineering activity to published DevRel content

- **Status:** In review
- **Author:** tech-lead · **Reviewers:** reviewer + enterprise-architect (cross-cutting) · **Approver:** tech-lead
- **Date:** 2026-07-29
- **Related:** [spec](../../../.specs/19_spec_eadros_devrel_os_summary.md) · ADR-0001…ADR-0016 ·
  [manifest](../../orchestrator/project.yaml) (`delivery_state.phase: design`) · milestones M2–M8

> Written *before* the code. Every claim below is sourced to an artifact in this repository; where
> this RFC **proposes** something the specification does not yet state, it is marked
> **PROPOSAL — needs approval** and is listed again under *Open items*. Nothing is inferred silently.

## Context

High-quality open-source projects fail from **absent distribution, not absent quality**. Maintainers
have no time for Developer Relations, and generic social-media tooling produces the promotional
register developer communities reject on sight
([PROBLEM_STATEMENT](../../../.specs/docs-eadros/vision/PROBLEM_STATEMENT.md)).

**What forces a decision now.** The specification is complete — 16 ADRs, the C4 views, the data
model, the state machine, the event contract, the intake interview and the verification strategy —
and **no implementation exists**: every `Impl` box in the
[roadmap](../../../.specs/docs-eadros/roadmap/MVP.md) is unchecked, and that is the accurate state.
Code begins at milestone M2 (*Foundation*). The delivery contract requires the design frozen,
reviewed and gated before it does, which is what this RFC is for.

**Three constraints shape the system, and none of them is "generate good text"** — text generation is
the commodity part:

1. **Most destinations forbid what is being automated.** Hacker News has no write API and bans by
   *domain*; Reddit's self-promotion rules are per-subreddit; LinkedIn gates posting behind app
   review. Over-automation can get the domain of the very project the tool was hired to promote
   permanently banned from the highest-signal channel in its ecosystem (ADR-0011).
2. **Detectably generated output falsifies the entire thesis** in one post, unrecoverably. The
   product is authenticity; a recognisable model register is a category failure, not a quality
   issue (ADR-0012).
3. **The pipeline reads private repositories, holds publishing credentials, and ingests text any
   stranger can write** on a public repository's pull request (ADR-0014).

**Fixed by the manifest.** TypeScript on Node, `project_kind: cli`, domain `software`, governance
posture `enterprise` (ADRs mandatory for security-relevant decisions), scope one repository per
program, ≤ 2 substantive posts per week, 6–8 channels.

## Decision

Eight load-bearing decisions. The first five consolidate what the ADRs settled; the sixth is a
maintainer decision recorded on 2026-07-29; the last two were settled during this review
(2026-07-29) and close open items O1 and O2. **D7 and D8 are interdependent** — the runtime floor
decides which SQLite bindings exist, and the binding's packaging decides the shape of the CI matrix
— so they are decided together rather than in sequence.

**D1 — The orchestrator is a fixed DAG with models inside the nodes, not an agent that plans the
pipeline.** Three of nine components hold a model — Angle, Copywriter, Reviewer — which is where
judgment genuinely lives: framing, writing, critique. The other six carry a **guarantee** rather
than a judgment, and a guarantee implemented in a prompt can only ever be an aspiration
([COMPONENTS](../../../.specs/docs-eadros/architecture/COMPONENTS.md)).

**D2 — The capability tier is derived from each destination's dated profile and stated to the
maintainer, never offered as a choice.** They choose *whether* to use a channel; the profile decides
*how*: `auto` dispatches on approval, `assisted` dispatches under quota and refuses when spent,
`draft` **has no dispatch path in code at all**. A profile older than 90 days blocks onboarding,
because a compliance rule that cannot be re-verified is silently wrong within a year (ADR-0011).

**D3 — Safety is a deterministic gate that runs twice, not a careful model.** `PrePublishGate`
evaluates eight stages — `secrets`, `deny_terms`, `paths`, `diff_cap`, `embargo`, `taint`, `claims`,
`voice_lint` — over **mined inputs before any model call** and over **final approved text before
dispatch**. `secrets` and `taint` have no off switch. Running on inputs is what stops injection:
filtering only outputs would mean the injected instruction had already executed with credentials in
hand. The input stages `paths`/`embargo`/`taint` run at **mining** time, so an ineligible commit
never becomes a candidate (ADR-0014, [mine](../../../.specs/docs-eadros/commands/mine.md)).

**D4 — One process, one SQLite file, no services.** WAL mode, `STRICT` tables, ULID `TEXT`
identifiers, ISO-8601 UTC timestamps, structured columns guarded by `json_valid()`. State, content,
approvals, outbox, transition ledger, metrics, event log and knowledge base (FTS5 + embeddings) all
live in that one file, and every event is written **in the same transaction** as the state change it
describes (ADR-0016, ADR-0002).

**D5 — Governance is enforced by the engine, not by the publisher's control flow.** Three schema
constraints carry the product's central claims structurally:

| Claim | Mechanism |
|---|---|
| No post exists without an approval | `posts.approval_id NOT NULL` |
| No post is published twice | `UNIQUE (posts.idempotency_key)` |
| An unedited approval cannot hide | `CHECK (mode = 'as-is' OR hash_before <> hash_after)` |

**D6 — `.eadros-core/` is the installation root.** Every asset the product installs is contained in
that directory: one directory to install, to upgrade, to inspect and to remove. The specification
already treats it as the movable unit — `upgrade` migrates `.eadros-core/` plus the manifest to a
newer core version ([upgrade](../../../.specs/docs-eadros/commands/upgrade.md)) — and this decision
makes the containment a property of the design rather than a convention. *Confirmed by the maintainer
on 2026-07-29; to be preserved through every later phase and recorded as an ADR (see Open items).*

**D7 — The runtime floor is Node 22 LTS; the CI matrix is Node 22 and 24 on all three platforms.**
*Settled in review, resolves O1.*

> **Provenance warning.** The support dates below are **not** from this repository and are not
> derivable from it. They come from the published Node.js release schedule as known to the drafting
> agent, whose knowledge has a cutoff; confirm them against `nodejs.org/en/about/previous-releases`
> before this RFC is approved. Everything else in D7 follows *from* them.

The manifest's profile baseline — Node 18/20/22 — is **stale at the date of this RFC**: Node 18 left
support on 2025-04-30 and Node 20 on 2026-04-30, so two of the three baseline cells exercise a
runtime that receives no security patches. For a tool that holds publishing credentials and reads
private repositories, testing on an unsupported runtime is a security-posture defect rather than
untidiness — and `governance.posture: enterprise` makes that an ADR-worthy call rather than a
janitorial one. The maintainer already recorded the profile values as *a baseline, not an irrevocable
decision* (manifest `toolchain:`, 2026-07-29); this is the design phase exercising exactly that.

- **Floor: Node 22 LTS** — the oldest release still receiving security support (to ~2027-04). Node 24
  is the newer LTS; the floor sits at 22 rather than 24 so the tool does not demand a runtime upgrade
  from users who are still supported. Raising it to 24 is a one-line change if you prefer a shorter
  support tail.
- **Matrix: {ubuntu-24.04, windows-2022, macos-14} × {Node 22, Node 24}** — six cells, replacing the
  baseline's five. The **full product** of OS × Node major is deliberate and is driven by D8: with a
  native addon in the dependency graph, `(OS, Node ABI)` *is* the risk surface, because a missing
  prebuild for one cell means a source compile on a user's machine and a blown cold-start budget.
  Without D8's addon this matrix would be over-provisioned; with it, it is the thing being tested.
- **TypeScript: 5.x, target ES2022, `module`/`moduleResolution: NodeNext`, `strict: true`** — the
  profile already treats strict `tsc` as the type-soundness gate, and this keeps that. ES2022 is
  retained because nothing in the design needs a newer target; the floor change is about the
  *runtime's support status*, not about language features.
- **Consequence, stated rather than buried:** users pinned to Node 20 are excluded. That is
  acceptable here — the install path is `npx`, and Node 20 is out of support.

**D8 — `better-sqlite3` for V1, with `node:sqlite` recorded as a migration target behind a testable
trigger.** *Settled in review, resolves O2.*

The store is not a preference question: D4 requires that **every event is persisted in the same
transaction as the state change it describes**, over a single-writer WAL connection.

- **A synchronous driver makes that transaction a lexical block.** An async driver makes it an
  interleaving hazard on the one connection the design allows — the failure mode is a half-written
  transaction under concurrency the store was never meant to have. `better-sqlite3` is synchronous by
  design, which is an architectural fit with D4, not a taste.
- **FTS5 is compiled into its bundled SQLite**, which `kb_fts` requires.
- **The embeddings BLOB cosine scan** is a tight read loop; a synchronous binding avoids paying
  per-row promise overhead on the hottest read path in the system.

**Its cost is real and is converted into a gate rather than assumed away.** `better-sqlite3` is a
**native addon**: installation resolves a prebuilt binary per `(OS, Node ABI)`, and when no prebuild
matches, npm compiles from source — which needs a local toolchain and will very likely breach the
*cold `npx` ≤ 60 s* budget. So the risk becomes a release gate, in the same spirit as D3:

> CI asserts, on a **clean image for every matrix cell**, that installation resolves a **prebuilt**
> binary and never invokes a compiler, and measures the cold `npx` run against the 60 s budget. A
> cell without a prebuild is a **release blocker**, not a warning.

**Migration trigger for `node:sqlite`** — stated so it is checkable rather than aspirational. The
built-in replaces the addon when all three hold: (a) the Node floor is at or above the first LTS
where `node:sqlite` is no longer experimental; (b) a probe confirms **FTS5 is compiled into Node's
bundled SQLite** on every matrix cell; (c) the replacement preserves the synchronous transaction
semantics D4 requires. Until then the driver sits behind the store module's own interface, so the
swap is one file rather than a refactor.

**(b) is genuinely unresolved and this RFC declines to assert it.** Whether Node's bundled SQLite
ships FTS5 is not determinable from this repository, and the drafting agent's knowledge of it is not
current enough to be load-bearing. Settle it by running the probe on each target version:

```bash
node -e "const {DatabaseSync} = require('node:sqlite'); const d = new DatabaseSync(':memory:'); try { d.exec('CREATE VIRTUAL TABLE t USING fts5(x)'); console.log('FTS5: available'); } catch (e) { console.log('FTS5: MISSING -', e.message); }"
```

Zero install cost makes `node:sqlite` the right long-term answer — which is why it is recorded as a
target with a trigger rather than dismissed.

### API contract (`api` / `systemdesign`)

The consumer surface is a **CLI**, aligned with spec §5 and
[commands/README](../../../.specs/docs-eadros/commands/README.md).

**Operations** — 23 commands in five groups (22 documented entries: `pause` / `resume` share one):

| Group | Commands | Boundary |
|---|---|---|
| Setup & lifecycle | `init`, `onboard [--recalibrate]`, `adopt`, `doctor`, `status`, `upgrade` | Drafts/reports; the human confirms |
| Positioning | `launch` | A **gate**, not a report: writes `positioning.last_audit`; campaigns declare a pre-condition on its score |
| Content pipeline | `mine`, `draft <id>`, `weekly`, `release`, `campaign <id>` | Drafts; **never publishes** |
| The gate | `review`, `approve --id`, `reject --id --reason`, `publish`, `retract --id`, `pause`, `resume` | `approve`/`reject` are **human-only — the agent may never call them** |
| Feedback | `metrics`, `digest`, `audit`, `eval` | Read-only, except `metrics` which writes only its own table |

**Payloads** — arguments are ids and reasons; behaviour is carried by flags. Global:
`--dry-run` (full pipeline against a mock publisher; the default for a program's first week),
`--explain` (scoring signals, the model routed per stage, and the token/cost estimate *before*
spending), `--budget-check` (refuses to start when the run would exceed `budget.cost_per_week`).
`mine` adds `--since <ref>`, `--all`, `--json`; `approve` adds `--as-is`, which is recorded.

**Error model** — the failure taxonomy consumers must handle. Every entry below is a refusal the
specification already states; the *exit-code mapping* is a proposal:

| Class | Raised when | Behaviour |
|---|---|---|
| `GateBlocked` | any `PrePublishGate` stage returns `block` | The subject does not advance; the verdict is persisted with the offending span |
| `ApprovalRequired` | dispatch attempted without an approval record | Refused by the schema, not by control flow (`approval_id NOT NULL`) |
| `ApprovalInvalid` | `edit-required` and the final text equals the draft | Refused unless `--as-is` is passed and recorded |
| `TierForbidden` | dispatch attempted on a `draft`-tier channel | **Unreachable by construction** — no dispatch path exists in code |
| `QuotaExhausted` | an `assisted` channel's quota is spent | Refuses; the payload is handed off rather than dropped |
| `PreconditionUnmet` | a campaign's `precondition` (e.g. `wave0_score`) fails | Refuses to start |
| `BudgetExceeded` | the run would cross `budget.cost_per_week` | Refuses under `--budget-check`; `pause` before the ceiling is crossed |
| `PolicyStale` | a channel profile is older than 90 days | Blocks onboarding for that channel |
| `CredentialProblem` | token absent, scope insufficient, or near expiry | Surfaced by `doctor` **before** it fails mid-campaign |
| `DispatchUnknown` | the process died between the platform call and the record | `outbox.outcome = 'unknown'`; `publish --reconcile` **asks the platform**, never retries blind |
| `RemovalImpossible` | `retract` on a channel whose `api.delete` is `unsupported` | Drafts a correction and says plainly that removal is impossible |

**Versioning — PROPOSAL, needs approval.** Three independent compatibility surfaces, only the second
of which the specification versions explicitly today:

1. **CLI surface** — SemVer on the package. MAJOR = removing or renaming a command or flag, or
   changing a default that changes what ships.
2. **Event contract** — the version is *in the type*: `dev.eadros.<domain>.<event>.v<N>`
   ([EVENTS](../../../.specs/docs-eadros/architecture/EVENTS.md)). A breaking change publishes `.v2`
   alongside `.v1`; it is never a silent reshape of `.v1`.
3. **Store schema** — a forward-only migration series (see the Data fold). MAJOR = a migration that
   is not forward-compatible with the previous release's reader.

`capabilities.api_spec` is **off**: EADROS exposes no HTTP surface, so a `docs/api/` OpenAPI stub
would describe nothing.

### Data & schema (`database`)

Within ADR-0004's frame: SQL is the **secondary** language declared at interview Q1.2
(`language.secondary_lang: sql`), never a primary profile.

**Entities** — 19 tables in five groups
([DATA_MODEL](../../../.specs/docs-eadros/architecture/DATA_MODEL.md)):

- *Pipeline* — `repos`, `candidates`, `campaigns`, `drafts`
- *Governance* — `claims`, `gate_verdicts`, `approvals`, `rejections`
- *Publishing* — `posts`, `outbox`, `retractions`, `post_transitions`
- *Measurement* — `post_metrics`, `repo_traffic`, `budget_ledger`
- *Knowledge & events* — `kb_documents`, `kb_fts` (FTS5 virtual), `kb_embeddings`, `events`

**Relations that carry the design.** A **campaign** and a **post** have separate lifecycles: one
story on four channels is one campaign and four posts, each with its own gate verdict, approval and
outcome — which is what makes a partial publish (live on one channel, awaiting a human on another,
failed on a third) representable rather than a contradiction between two status columns.
`retractions.gate_verdict_id` points at *the verdict that wrongly passed the content*, closing the
learning loop.

**Normalization — the target form is a PROPOSAL, needs approval.** The four denormalizations below
are each stated by the specification ([DATA_MODEL](../../../.specs/docs-eadros/architecture/DATA_MODEL.md)
§"Rules are denormalised at the moment they were applied"); the *target normal form* is not, and this
RFC proposes **third normal form** as the baseline the deviations are measured against. Each
deviation exists because an audit that explains last month's post using this month's rules is not an
audit:

| Denormalized | Reason |
|---|---|
| `drafts.voice_profile` | The profile **as applied**, not as it is now |
| `posts.tier_at_publish` | The tier in force at dispatch |
| `posts.policy_verified_on` | The channel profile's verification date, as applied |
| `posts.state` / `campaigns.state` | A cache of the last `post_transitions` row — the ledger is the fact |

Append-only throughout: approvals, gate verdicts, transitions, metrics and events are never updated
in place. `post_metrics.is_gap` / `repo_traffic.is_gap` mark a missed snapshot, so an absent
measurement never renders as a zero — the GitHub Traffic API's ~14-day window makes the loss
permanent.

**Migration policy — PROPOSAL, needs approval.** The specification names *"SQLite store, schema and
migrations"* (roadmap M1.3 → manifest 2.3) but does not state the policy. Proposed: **forward-only,
numbered migrations**, each in its own transaction, with the applied series recorded in the store;
`user_version` (or an equivalent table) carries the schema version; no down-migrations — a mistaken
migration is corrected by a new forward migration, because a down-migration that has already lost
data is a fiction. `upgrade` proposes the diff; the human applies it.

### Scalability budgets (`scalability`)

The domain profile `software` declares **no `hard_budget: true` NFR axes**, so the `nfr-budgets`
audit gate has nothing it is obliged to enforce. That is a property of the domain, not permission to
be vague: the numbers below are stated as the reference the `audit` command and the `eval` cost suite
check against, and they are the manifest's `spec.nonfunctional_reqs` verbatim.

| Budget | Target | Measurement |
|---|---|---|
| Model calls during mining | **exactly 0** | `eval` cost suite |
| Cost per published post | ≤ **€0.15** | `budget_ledger`, per stage |
| Mining latency | 500-commit window scored ≤ **10 s** | miner benchmark |
| Candidate → review queue | p95 ≤ **3 min** | pipeline timestamps |
| Human review time per post | ≤ **5 min** | review-queue timestamps |
| Leak-gate recall on secrets | **1.0** — absolute | adversarial suite, hard gate |
| Prompt-injection containment | **1.0** — absolute | adversarial suite, hard gate |
| Double-publish rate | **0** — absolute | property test + `posts.idempotency_key` |
| Gate false-positive rate | ≤ **0.10** | `gate_verdicts.false_positive`, via `audit` |
| `auto`-tier publish success | ≥ **99%** excl. platform outages | outbox outcomes |
| Metrics snapshot gap rate | ≤ **1%** of days | `post_metrics.is_gap` |
| Install cost | **0 services**; cold `npx` ≤ **60 s** | CI on a clean image |
| Installation containment | **every installed asset under `.eadros-core/`** | post-install file-set assertion on a clean image (D6) |

Three are absolutes rather than trade-offs — secrets recall, injection containment, double-publish —
and each sits in deterministic code **precisely so it can be gated at 100%**.

### Algorithm sketch (`pseudocode`)

The miner is the one non-obvious core algorithm: it is the system's real intellectual property, and
it is the whole cost bound. Language-free, from
[mine](../../../.specs/docs-eadros/commands/mine.md):

```
mine(window):
    activity  <- repository activity since pipeline_state.last_mine_at (or --since)
    eligible  <- gate.input_pass(activity)          # paths, embargo, taint — HERE, not later
    candidates <- []
    for item in eligible:
        if item matches a hard exclusion:            # dep bumps, lockfiles, format-only,
            record(score 0, excluded_reason); continue   # merges, generated, vendored
        score   <- SUM(weight[s] for each signal s present in item)   # config/scoring.yaml
        shapes  <- archetypes proposed by the present signals
        allowed <- shapes INTERSECT voice.archetypes  # consent filters the RANKING, not the output
        if allowed is empty:
            record(score, excluded_reason = 'no consented archetype'); continue
        hash    <- content_hash(item.source_refs)
        verdict <- 'duplicate'  if hash seen
                   'supersedes' if similar to a past post above threshold AND the work moved on
                   'new'        otherwise
        if verdict = 'duplicate': record(score, excluded_reason); continue
        candidates.append({item, score, signals incl. zeroed, allowed, hash, verdict})

    rank candidates by score, descending
    kept, cut <- top budget.max_campaigns_per_run, the remainder
    PERSIST kept AND cut                     # the rejection set is the eval recall corpus
    REPORT  kept AND cut with their scores   # a ranker whose rejections are invisible cannot be corrected
    pipeline_state.last_mine_at <- now       # nothing else is written

INVARIANTS
    no model call on any path — a tie broken by a model has broken the command
    identical input yields identical output — or the cost model has no floor
    a hard-excluded item never scores above zero — asserted by its own regression corpus
```

### Cross-cutting

**Security.** The threat model (spec §7) pairs eight threats with the control that answers each:
prompt injection (taint on ingestion, propagated to derived events; untrusted spans passed as
delimited **data**, never as instruction; containment asserted at the boundary), confidential leak
(unconditional secret scanning, deny-terms, path allowlist — private repos start **deny-all** —
diff-line cap, embargo), self-inflicted platform ban (the tier model; an upward override requires a
recorded ADR quoting `ban_scope`), credential theft (the manifest stores credential *locations*,
never values; minimum scopes; expiry surfaced by `doctor`), double publish (outbox commits intent
before the call; timeouts **reconcile**, never retry blind), supply chain (no public plugin API
before V1), reputational damage (`retract` runbook, kill switch, the paused queue re-gated under
updated rules before resuming), unbounded spend (deterministic pre-filter, tier routing, iteration
cap, weekly ceiling with `pause` before it is crossed).

**Enterprise posture.** `governance.posture: enterprise` makes an ADR **mandatory** for
security-relevant decisions. On this design that is not a formality: D2, D3, D5 and D6 each change
the security surface, and each already has or requires an ADR.

**Observability.** Every event is persisted with `correlation_id` and `causation_id`, so a single
query returns the complete account of how one CI failure became three posts — who approved it, what
the gate checked, what it cost, what it earned.

**Performance.** The knowledge base is FTS5 plus an **exhaustive cosine scan** over an embeddings
table in the same file: at a few thousand chunks this completes in microseconds, and a dedicated
vector service would be answering a question this scale never asks.

## Alternatives

| # | Alternative | Why rejected |
|---|---|---|
| A1 | An agent that **plans the pipeline** | Adds non-determinism, unbounded cost and an unauditable trail to a system whose entire value is governance. A fixed DAG makes the cost a function of the top-K cut, not of repository activity |
| A2 | **Six specialised agents**, as ADR-0003 proposed | Four of the six turned out to carry a *guarantee*, not a judgment — reproducible ranking, recall 1.0 on planted secrets, never double-post, never exceed a ceiling. Each is exact, free and testable in code, and unprovable in a prompt at any budget |
| A3 | Let the **maintainer choose the automation tier** | The platform's terms decide what is lawful, not the user's preference. A wrong choice here can get the promoted project's domain **permanently banned** from the highest-signal channel in its ecosystem (ADR-0011) |
| A4 | **Human review as the safety control**, as ADR-0008 claimed | Reviewers rubber-stamp by the third week; attention is not a control. Human review is retained as the *authorisation* gate and replaced as the *safety* gate by deterministic code (ADR-0014) |
| A5 | Filter **model outputs only** | By the time an injected instruction appears in an output, it has already executed with credentials in hand. Containment requires the input pass |
| A6 | **Postgres + ChromaDB + a LiteLLM sidecar** | A tool whose stated user is a maintainer with no time cannot ask for five processes before it does anything. No embedded Chroma build exists for this runtime, and the corpus is hundreds of documents (ADR-0016) |
| A7 | Ship a **public plugin API** now | ADR-0004 committed to one before three adapters exist. Third-party code running beside publishing credentials needs a sandbox first; the honest sequence is an internal adapter interface now, a public API when third-party demand is real |
| A8 | Spread installed assets across home/config/cache directories, per-OS convention | Defeats D6: no single unit to upgrade, inspect or remove, and `upgrade` already treats `.eadros-core/` as *the* movable unit. The per-OS convention buys tidiness and costs auditability — the wrong trade for a tool holding publishing credentials |
| A9 | A **web dashboard** as the review surface for V1 | A genuinely different deployment with a different store. The review surface stays pluggable so the CLI is not a dead end, but V1 ships the CLI |
| A10 | Keep the profile baseline matrix, **Node 18/20/22** | Node 18 and 20 are both out of support at the date of this RFC, so two of three cells would certify the tool on a runtime receiving no security patches — for a process holding publishing credentials, that is a control failure, not a housekeeping item (D7) |
| A11 | **`node:sqlite`** (the built-in) for V1 | Rejected *for V1*, not on principle — it is the recorded migration target. Two blockers: it needs the floor D7 only just set, and **FTS5 in Node's bundled SQLite cannot be asserted from this repository**. Adopting it on an unverified FTS5 assumption would put the knowledge base — `kb_fts` — on a bet |
| A12 | A **WASM** build (`node-sqlite3-wasm`, `wa-sqlite`) | Removes the native addon and is perfectly portable, but weakens the file-I/O and WAL story and taxes the embeddings scan, which is the hottest read path. It buys install portability with the one performance budget the KB depends on — and D8 already converts the addon's install risk into a measured gate |
| A13 | **`sql.js`** | In-memory with manual persistence. Incompatible with a single-file WAL store that must survive a process death mid-outbox — exactly the `DispatchUnknown` path that reconciles rather than retrying blind (D4) |
| A14 | **`libsql`** | Adds a SQLite *fork* to a system whose store decision (ADR-0016) exists to minimise install surface. Its replication features serve the multi-repository deployment this design explicitly defers |

## Consequences

**What this makes easy.** Cost is bounded by construction, because the only stage that scales with
repository activity holds no model. Safety claims are test results rather than hopes. A partial
publish is one honest row set rather than a contradiction. `SELECT * FROM events WHERE
correlation_id = ?` answers "how did this happen" completely. `mine` is free, so the ranking can be
inspected and tuned before anything is paid for. One directory installs, upgrades and uninstalls the
system (D6).

**What this makes harder.** Every new channel needs a **dated** profile and a human re-verification
on a 90-day clock — the tool cannot read a terms-of-service page and understand it. `draft`-tier
channels mean permanent manual work on the highest-signal destination, and that is the correct
answer rather than a gap. There is no unattended publishing path at all, by design. The single-writer
store caps concurrency, and a multi-repository workspace is a **different deployment**, not a
configuration flag. Voice authenticity is settled by a human calibration loop, not a metric. The
Node 22 floor (D7) excludes users pinned to an out-of-support runtime, and the native addon (D8)
makes **prebuild availability a per-cell release gate** — six cells to keep green, and a missing
prebuild blocks a release rather than degrading quietly to a source compile.

**Migration path.** None required: no implementation and no users exist. Work starts at manifest
milestone **M2 — Foundation** (manifest, store, event bus, state machine, profile loaders), which is
the prerequisite for every later milestone.

**Follow-up roadmap items.** M2 Foundation → M3 Wave 0 positioning → M4 Mining and drafting → M5 Gate
and publish (`auto`) → M6 `assisted` and `draft` tiers → M7 Measurement → M8 Verification harness —
38 items, recorded in the manifest with their source numbering.

**Open items — decisions this review must settle.** Each is flagged because the repository does not
determine it, and this RFC declines to invent it:

| # | Open item | Note |
|---|---|---|
| ~~O1~~ | ~~Node / TypeScript version floor and the CI matrix~~ | **Settled in review, 2026-07-29 → D7.** Floor Node 22 LTS; matrix {ubuntu, windows, macos} × {22, 24}; TS 5.x / ES2022 / NodeNext / strict. Carries one dependency on information outside this repository — the Node support dates — flagged in D7 |
| ~~O2~~ | ~~SQLite driver — `node:sqlite` vs `better-sqlite3` vs another~~ | **Settled in review, 2026-07-29 → D8.** `better-sqlite3` for V1 on the synchronous-transaction argument; prebuild availability becomes a per-cell release gate; `node:sqlite` recorded as a migration target behind a three-part trigger, one part of which (**FTS5 in Node's bundled SQLite**) is an open verification with a probe supplied |
| O3 | **Migration policy** | Proposed above (forward-only, numbered, versioned); not stated in the specification |
| O4 | **Exit-code taxonomy** for the error model | The refusals are specified; their CLI exit codes are not |
| O5 | **Concrete `.eadros-core/` layout** | D6 fixes the containment; the internal layout and an ADR recording D6 are still to be written |
| O6 | Roadmap items the specification itself marks `○`/`◐` | 2.7 provider abstraction, 2.8 `config/` overlay, 4.5 knowledge base, 6.4 further channel profiles, 6.5 GitHub-native surface |

## Approval

Filled by the approver **after** review (this is the `rfc-approved` gate's record):

```
approved-by: tech-lead (2026-07-29)
```

Reviewers (structured findings addressed): reviewer — resolved · enterprise-architect
(cross-cutting) — resolved.

Review record (2026-07-29) — every load-bearing claim was re-verified against the specification
rather than accepted from the draft. Four findings, all resolved in this revision:

| # | Finding | Resolution |
|---|---|---|
| F1 | The Data fold claimed **17 tables**; `DATA_MODEL.md` defines **19** `CREATE TABLE`s, and the RFC's own enumeration two lines below listed 19 (4+4+4+3+4) | Corrected to 19 |
| F2 | **22 commands** is the row count of `commands/README.md`; one row carries two commands (`pause` / `resume`), so the surface is 23 | Corrected to "23 commands … 22 documented entries" |
| F3 | **Third normal form** was asserted as fact, but no specification artifact states a target normal form — contrary to this RFC's own header rule that anything beyond the spec is marked *PROPOSAL* | Marked as a proposal; the four denormalizations, which *are* sourced, kept as stated |
| F4 | `Status: Draft` while the document was entering review | Set to *In review* |

Verified and CONFIRMED without change: the eight `PrePublishGate` stages, named exactly as in
ADR-0014; D5's three schema constraints, present verbatim at `DATA_MODEL.md:133,155,166`; D3's claim
that `paths`/`embargo`/`taint` run at **mining** time, with the same reasoning at `mine.md:17-18`;
all four denormalizations (`DATA_MODEL.md:19,74,158-159,170,214`); and that the `software` domain
declares no `hard_budget: true` axis, so the `nfr-budgets` gate is genuinely vacuous here rather
than being waved through.

**Open items O1 and O2 were settled during this review**, at the maintainer's instruction, as **D7**
(runtime floor and CI matrix) and **D8** (SQLite driver). Both are marked *PROPOSAL* in the Decision
section: they are the RFC proposing what the repository does not determine, which is what an RFC is
for — not imported fact.

Two consequences must be carried out **after** approval, and are named here so they are not lost:

1. **Manifest sync.** D7 supersedes the profile baseline the maintainer explicitly recorded as
   revisable. On approval, `toolchain.lang_standard`, `ci.tier1_platforms`, `ci.matrix` and
   `ci.setup_steps` are updated to the D7 values by the **`enterprise-architect` as state writer** —
   `authority.yaml` grants `orchestrator/project.yaml` to that role alone, so the `tech-lead`
   authoring this RFC may not perform it (#346, ADR-0025).
2. **Two verifications this RFC cannot perform**, both cheap and both blocking the code they feed —
   not the approval of this document:
   - the Node support dates underpinning D7, against the published release schedule;
   - FTS5 availability in Node's bundled SQLite, via the probe in D8, on each target version.

## References

- [Specification summary](../../../.specs/19_spec_eadros_devrel_os_summary.md) — the six-section source
- Architecture: [SYSTEM_CONTEXT](../../../.specs/docs-eadros/architecture/SYSTEM_CONTEXT.md) ·
  [CONTAINERS](../../../.specs/docs-eadros/architecture/CONTAINERS.md) ·
  [COMPONENTS](../../../.specs/docs-eadros/architecture/COMPONENTS.md) ·
  [DATA_MODEL](../../../.specs/docs-eadros/architecture/DATA_MODEL.md) ·
  [EVENTS](../../../.specs/docs-eadros/architecture/EVENTS.md) ·
  [STATE_MACHINE](../../../.specs/docs-eadros/architecture/STATE_MACHINE.md)
- Decisions: [ADR-0011 tiers](../../../.specs/docs-eadros/adr/ADR-0011-channel-capability-tiers.md) ·
  [ADR-0012 voice](../../../.specs/docs-eadros/adr/ADR-0012-voice-profile-and-calibration.md) ·
  [ADR-0013 cost](../../../.specs/docs-eadros/adr/ADR-0013-cost-control-and-model-routing.md) ·
  [ADR-0014 gate](../../../.specs/docs-eadros/adr/ADR-0014-deterministic-pre-publish-gate.md) ·
  [ADR-0015 attribution](../../../.specs/docs-eadros/adr/ADR-0015-attribution-methodology.md) ·
  [ADR-0016 store](../../../.specs/docs-eadros/adr/ADR-0016-local-first-single-file-store.md)
- Verification: [eval strategy](../../../.specs/docs-eadros/eval/README.md)
- Roadmap: [MVP](../../../.specs/docs-eadros/roadmap/MVP.md)
- Command surface: [commands/README](../../../.specs/docs-eadros/commands/README.md)
