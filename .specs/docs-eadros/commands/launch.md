# `/eadros launch` — the Wave 0 positioning audit

Audits whether the repository explains itself, and writes `positioning.last_audit`. Owned by the
**devrel-architect** role.

**This is a gate, not a report.** Campaigns declare preconditions on its score, and a campaign whose
precondition is unmet does not fire ([campaign](campaign.md)). Distributing a project whose README
does not explain itself converts the one launch you get into a bounce — the audit exists so that
outcome requires a deliberate override rather than an oversight.

## Procedure

### 1. Deterministic checks — free, exact, and most of the value

These run without a model. They are cheap enough to run on every commit and they catch the failures
that actually lose readers:

| Check | Pass condition |
|---|---|
| Demo above the fold | Image, GIF or video link within the first screen of the README |
| Install command visible | A runnable command in the first screen, not below a features table |
| First 400 characters | Describe the **problem solved**, not the technology stack |
| Repository description | Set, and not a restatement of the name |
| Repository topics | ≥ 3, drawn from `positioning.keywords` |
| Licence | Present and SPDX-identified |
| Keywords present | `positioning.keywords` appear in description, topics and README |
| Docs entry point | A linked docs page or a `docs/` directory |
| Latest release | Exists, and is not older than the last 20 commits |

**Repository topics and description are the cheapest discoverability lever available** — fully
API-supported, zero terms-of-service risk, and where GitHub's own search actually looks. They fail
this audit more often than anything else on the list, because nobody thinks of them as marketing.

### 2. Rubric-scored review — a model, on the parts that need judgment

Three questions, each scored against a stated rubric rather than a vibe:

- **Ten-second clarity.** From the first screen alone, can a reader state what this is and who it is
  for? Scored against `program.audience`, not against general interest.
- **Differentiation.** Held against `positioning.comparables` and `positioning.differentiator`. The
  test is the one from the interview: *could a competitor's README also claim this?* If yes, it is
  not a differentiator.
- **Proof.** Is `positioning.proof_assets` actually reachable from the README — a benchmark, a demo,
  a diff, a decision a reader can check?

Comparables live in the manifest rather than in the prompt, because a hardcoded competitor list rots
within a quarter.

### 3. Score and record

A 0–100 score plus `blocking[]` findings, written to `positioning.last_audit` with a timestamp.
Deterministic checks dominate the weighting: they are objective, and a project that fails them is
failing at something it can fix this afternoon.

### 4. Report, and change nothing

Output is the score, the failed checks, and the specific edit each one needs. **`launch` never
modifies the repository** — not the README, not the topics, not the description. A tool that
silently rewrites a maintainer's front page has misunderstood whose project it is.

## Interaction with campaigns

```yaml
campaigns:
  - id: show-hn-v1
    precondition: "positioning.last_audit.score >= 80 and positioning.demo_ref != ''"
```

`campaign` evaluates the precondition against the **recorded** audit. An audit older than 30 days is
treated as absent — the README has moved on, and a stale pass is not a pass.

## Boundary

Read-only. Reports a score and blocking findings; changes no files, contacts no platform, publishes
nothing. The maintainer decides what to fix, and re-runs.

## Flags

| Flag | Effect |
|---|---|
| `--deterministic-only` | Skip the rubric pass; free, and enough for most re-runs |
| `--explain` | Show the rubric, the evidence quoted, and the per-check weighting |
| `--json` | Machine-readable, for a CI positioning check |
