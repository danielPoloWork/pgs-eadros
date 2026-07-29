# `/eadros mine` — rank repository activity into story candidates

Scores repository activity and prints ranked candidates. Owned by the **story-miner** component —
deterministic code, not an agent ([agents/README](../agents/README.md)).

**`mine` makes no model calls.** That is its defining property, not an optimisation: it is what
decouples cost from repository activity ([ADR-0013](../adr/ADR-0013-cost-control-and-model-routing.md)),
and it is what lets a maintainer inspect the ranking for free before paying for the pipeline that
follows it. A `mine` implementation that reaches for a model to break a tie has broken the command.

This is also the system's real intellectual property. Everything downstream is competent text
generation; deciding **which commit deserves a story** is the part nobody else does well.

## Procedure

1. **Load and filter.** Read activity since `pipeline_state.last_mine_at` (or `--since`). Apply the
   [pre-publish gate](../adr/ADR-0014-deterministic-pre-publish-gate.md) `paths`, `embargo` and
   `taint` stages **here**, not later: an ineligible commit must never become a candidate, and
   filtering at mining time is free while filtering after drafting is not.

2. **Score.** Each candidate accumulates weighted signals. Weights live in
   `config/scoring.yaml` so they can be tuned and evaluated rather than argued about:

   | Signal | Why it carries a story |
   |---|---|
   | CI gate intercepted a change | A real defect was prevented, with a log to prove it — the strongest signal available |
   | Revert or hotfix | Something went wrong and was fixed; the honest post-mortem shape |
   | An ADR was added or amended | A decision with a stated cost, already written down |
   | An architectural boundary was crossed | The change was structural, not incremental |
   | Large net deletion | Removed complexity is the most under-told and best-received story shape |
   | Measured benchmark delta | A number a reader can check |
   | Issue closed with a linked bug | Problem → cause → fix, already structured |
   | First-time external contributor | A community event, and the only one that is time-sensitive |
   | Release published | Scheduled, and handled by `release` rather than here |

   **Hard exclusions**, scored zero and never surfaced: dependency bumps, lockfiles, formatting-only
   changes, merge commits, generated files, vendored directories, and any change whose diff is
   entirely in paths outside `safety.path_allowlist`.

3. **Match archetypes, filtered by consent.** Each signal proposes story shapes, then
   `voice.archetypes` filters them ([ADR-0012](../adr/ADR-0012-voice-profile-and-calibration.md)).
   A repository whose maintainer left `postmortem: false` never sees revert-driven candidates
   offered as post-mortems — the consent decision propagates into the ranking rather than being
   enforced downstream, so the maintainer is not repeatedly shown content they already declined.
   A candidate whose every archetype is disabled is dropped, with the reason recorded.

4. **Deduplicate.** Two checks, both stated rather than implicit: a **content hash** over the
   candidate's source refs, and a **similarity check against the past-post index** above a
   configured threshold. Without this, the weekly run re-proposes the same story until someone
   notices. Near-duplicates are shown as *superseding* a prior post when the underlying work moved
   on — that is a legitimate follow-up, not a repeat.

5. **Rank and cut.** Sort by score; keep the top `budget.max_campaigns_per_run`. **Print the ones
   that were cut, with their scores** — a ranker whose rejections are invisible cannot be corrected,
   and the cut list is where a maintainer notices the weights are wrong.

6. **Report.** For each candidate: id, score, the contributing signals with their weights, the
   source refs, the eligible archetypes, and the dedup verdict. `--explain` adds the full signal
   breakdown including zeroed ones.

7. **Write state.** Update `pipeline_state.last_mine_at`. Candidates persist so `/eadros draft <id>`
   can pick one up; nothing else is written.

## Verification

The ranker is the one component with an obvious golden-set eval, and `/eadros eval` runs it: a
labelled corpus of real commits — *worth a story* / *not worth a story* — scored for
**precision and recall**, tracked across weight changes. This is what makes tuning an experiment
rather than a preference.

The exclusion list gets its own regression corpus: a dependency bump that scores above zero is a
test failure, not a judgment call.

## Boundary

Read-only and free. `mine` writes no content, calls no model, contacts no platform, and spends
nothing. It is safe to run on every commit, and safe to run repeatedly while tuning weights — which
is the point.

## Flags

| Flag | Effect |
|---|---|
| `--since <ref>` | Override the window; defaults to `pipeline_state.last_mine_at` |
| `--explain` | Full signal breakdown per candidate, including zeroed signals and exclusion reasons |
| `--all` | Show every candidate, not just the top K |
| `--json` | Machine-readable output for the eval harness |
