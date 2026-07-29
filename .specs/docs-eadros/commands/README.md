# EADROS command surface

The command set follows the EADOS two-layer convention exactly:

- **Canonical procedure** — `commands/<name>.md` in this directory. The single source of truth.
- **Thin host adapter** — `.claude/commands/eadros/<name>.md` in the target repository. An adapter
  **surfaces** a command; it never adds behaviour (EADOS ADR-0019 class 4). It names the procedure
  file, the acting role, and passes `$ARGUMENTS`. Adapters are generated per host, never
  hand-written per host.

Every command below states its **boundary**: what the agent drafts versus what the human owns.
That line is the product. A command that crosses it silently is a defect, not a convenience.

## The set

### Setup & lifecycle

| Command | What it does | Boundary | MVP |
|---|---|---|---|
| [`/eadros init`](init.md) | Frames the program: interview [Phase 0–1](../orchestrator/interview.md), imports the EADOS manifest, writes the `devrel.yaml` skeleton | Drafts the manifest; human confirms | ✅ |
| [`/eadros onboard`](onboard.md) | The channel + voice interview ([Phase 2–5](../orchestrator/interview.md)), ending in the voice calibration loop. `--recalibrate` re-runs Phase 3 alone | Human answers, confirms, and owns every tier acknowledgement | ✅ |
| [`/eadros adopt`](adopt.md) | Brownfield intake: a repo that **already** posts. Read-only presence map → goal menu → `adoption:` block | Read-only; proposes, never migrates | |
| [`/eadros doctor`](doctor.md) | Preflight: tokens present, scopes sufficient, quota remaining, credentials near expiry, channel policies older than 90 days | Reports; fixes nothing | ✅ |
| [`/eadros status`](status.md) | Where every campaign sits in the state machine, budget consumed, review queue depth | Read-only | ✅ |
| [`/eadros upgrade`](upgrade.md) | Migrates `.eadros-core/` and the manifest to a newer core version | Proposes a diff; human applies | |

### Positioning

| Command | What it does | Boundary | MVP |
|---|---|---|---|
| [`/eadros launch`](launch.md) | The **Wave 0** audit: deterministic README/repo checks plus a rubric-scored positioning review against `positioning.comparables`. Writes `positioning.last_audit` | Reports a score and blocking findings; changes no files | ✅ |

`launch` is a **gate, not a report**: a campaign whose `precondition` names `wave0_score` will not
fire until the audit passes. This is the doctrine of Wave 0 made executable — positioning before
distribution, enforced rather than recommended.

### Content pipeline

| Command | What it does | Boundary | MVP |
|---|---|---|---|
| [`/eadros mine`](mine.md) | **Deterministic story scoring only — no model calls.** Ranks repository activity and prints scored candidates with their signals | Read-only, free, inspectable | ✅ |
| [`/eadros draft <id>`](draft.md) | Runs the agent pipeline for one mined candidate through to the review queue | Drafts; never publishes | ✅ |
| [`/eadros weekly`](weekly.md) | The cadence run: mine → rank → draft within `cadence.weekly` and every channel ceiling | Drafts; never publishes | ✅ |
| [`/eadros release`](release.md) | Fires on `release.published`: mines release notes, breaking changes, trade-offs | Drafts; never publishes | |
| [`/eadros campaign <id>`](campaign.md) | Runs a milestone campaign **after evaluating its `precondition`** | Refuses when the precondition is unmet | |

Separating `mine` from `draft` is deliberate and load-bearing. Mining is the system's real
intellectual property and it costs nothing to run; drafting is where the money goes. A maintainer
who can inspect the ranking for free will trust the pipeline that follows it — and a bad ranking
gets caught before it has been paid for.

### The gate

| Command | What it does | Boundary | MVP |
|---|---|---|---|
| [`/eadros review`](review.md) | Opens the human review queue on the configured surface | The human reads | ✅ |
| [`/eadros approve --id <id>`](review.md) | Records approval. Under `approval_mode: edit-required` it is **rejected unless the final text differs from the draft**, or `--as-is` is passed and recorded | **Human only.** The agent may never call this | ✅ |
| [`/eadros reject --id <id> --reason`](review.md) | Records rejection with a reason; the reason feeds the eval corpus | Human only | ✅ |
| [`/eadros publish`](publish.md) | Dispatches approved content. `auto` publishes; `assisted` publishes under quota; **`draft` never dispatches** — it emits the payload, the composer link and the channel checklist for a human | Agent dispatches only what a human approved, and only where the tier permits | ✅ |
| [`/eadros retract --id <id>`](retract.md) | The wrong-post runbook. Branches on the channel's `api.delete`: `supported` → delete; `edit-only` → unpublish or correct in place; `unsupported` → draft a correction and tell the maintainer plainly that removal is impossible | Proposes; the human confirms every destructive step | ✅ |
| [`/eadros pause` / `resume`](pause.md) | The kill switch. `pause` blocks every dispatch immediately; **anyone may pause, only `governance.kill_switch_owner` may resume**, and resume re-gates the held queue | Either party may pause; only the owner resumes | ✅ |

`retract` exists because the alternative is discovering the runbook during the incident. It is also
why `api.delete` is a required field on every channel profile: a channel where retraction is
impossible — Hacker News — must be known to be strictest *before* publishing, not after.

### Feedback

| Command | What it does | Boundary | MVP |
|---|---|---|---|
| [`/eadros metrics`](metrics.md) | The **daily** traffic snapshot. Append-only | Automated; writes only its own table | ✅ |
| [`/eadros digest`](digest.md) | The maintainer summary: what shipped, what it cost, what moved | Read-only | |
| [`/eadros audit`](audit.md) | Governance audit: budget adherence, leak-gate false-positive rate, tier compliance, stale channel policies, approvals that used `--as-is` | Read-only | |
| [`/eadros eval`](eval.md) | Runs the [verification suites](../eval/README.md) — miner ranking, reviewer per-class detection, injection containment, leak recall, channel contracts, cost regression | Read-only | |

`metrics` is not optional and not on-demand. The GitHub Traffic API retains roughly **14 days**;
a snapshot missed is data lost permanently, and the whole analytics story depends on a series
nobody can reconstruct later.

## Global flags

| Flag | Effect |
|---|---|
| `--dry-run` | Runs the full pipeline against a mock publisher. **The default for a program's first week.** |
| `--explain` | Prints the scoring signals, the model routed to each stage, and the token/cost estimate before spending anything |
| `--budget-check` | Refuses to start when the run would exceed `budget.cost_per_week` |

## What no command does

There is no `/eadros auto` and no daemon that publishes unattended. Every path to a live post runs
through `approve`, and `approve` is a human verb. That constraint is not a limitation of the
implementation — it is the reason the output is worth reading.
