# ADR-0004: Installation is non-invasive — a closed allowlist, not a promise

- **Status:** Accepted
- **Date:** 2026-07-29
- **Deciders:** Maintainer
- **Related:** RFC-0001 D6; ADR-0003; roadmap items 1.10, 2.2, 6.5

## Context

EADROS is a **plugin**. It is installed into repositories it does not own, whose maintainers did
not ask for their project's shape to change. The specification already states this doctrine, in
its own voice, about the one command most tempted to violate it —
[`launch`](../../../.specs/docs-eadros/commands/launch.md) reports a positioning score and
changes no files, because a tool that *"silently rewrites a maintainer's front page has
misunderstood whose project it is"*.

RFC-0001 **D6** committed to containment: every asset the product installs lives under
`.eadros-core/`. But **D6 enumerates no exceptions**, and an invariant without its exception list
cannot be checked — the item-1.10 post-install assertion has no way to tell a violation from a
permitted write. In practice the product already writes outside `.eadros-core/`, in three places
that were never reconciled with D6:

1. **The user's manifest lands in a new root directory.**
   [`init.md`](../../../.specs/docs-eadros/commands/init.md) step 4 and
   [`questionnaire.yaml`](../../../.specs/docs-eadros/orchestrator/questionnaire.yaml) `meta.output`
   both specify `orchestrator/devrel.yaml`. Installing EADROS would create an `orchestrator/`
   directory in the host project. Specified, not accidental.
2. **A roadmap item plans to edit the host README.** Item 6.5, *GitHub-native surface — topics,
   description, README badges*, writes into a file the maintainer owns.
3. **Host command adapters live at a fixed path outside the bundle.**
   `.claude/commands/eadros/<name>.md`, because the host reads that exact location and no other.

The third is unavoidable. The first two are not. What makes all three a problem today is the same
thing: the boundary is asserted in prose and enforced nowhere.

## Decision

**Installation writes inside `.eadros-core/`, plus a closed allowlist of paths that external
systems fix. It never modifies a file that already exists.**

**The allowlist is closed. Adding to it requires superseding this ADR.**

| Path | Why it cannot live in the bundle |
|---|---|
| `.eadros-core/**` | The installation root itself (D6) |
| `.claude/commands/eadros/**` (and the equivalent per host) | The host reads command adapters from a fixed path and no other. **Create-only**: EADROS writes files under its own `eadros/` subdirectory and never touches a sibling |
| `.gitignore` | **Append-only, one line**, and only with consent. The alternative is the user's repository tracking a database and credentials-adjacent state |

Three rules make the boundary real rather than aspirational:

- **No pre-existing file is ever modified.** Not `README.md`, not `AGENTS.md`, not `CLAUDE.md`,
  `GEMINI.md`, `ROADMAP.md`, `CHANGELOG.md`, `SECURITY.md`, not `package.json`, not any file the
  host project already had. The single exception is the consented one-line `.gitignore` append,
  which is an append and not a rewrite.
- **The manifest moves inside the bundle**: `orchestrator/devrel.yaml` becomes
  **`.eadros-core/devrel.yaml`**. No `orchestrator/` directory is created in a host project.
- **Anything EADROS wants changed in a file it does not own, it proposes.** It prints the diff
  and the maintainer applies it. This is exactly `launch`'s existing posture, generalized from one
  command to the whole product.

**Uninstallation is `rm -rf .eadros-core/` plus removing the two allowlisted paths.** Stated as a
property because it is testable, and because a product that cannot be cleanly removed was never
really contained.

**Enforcement is a gate, not a review.** Item 1.10 becomes a file-set assertion that installs into
a *populated* fixture repository on a clean image, then fails on **any** path outside the allowlist
and on **any** checksum change to a pre-existing file. The reasoning is [ADR-0014](../../../.specs/docs-eadros/adr/ADR-0014-deterministic-pre-publish-gate.md)'s,
applied to a second surface: a guarantee that depends on a reviewer remembering it is a guarantee
that decays. A containment rule policed by prose is worth what the prose is worth.

## Alternatives

| # | Alternative | Why rejected |
|---|---|---|
| A1 | **Leave D6 as prose and trust review** | The three leaks above all passed review, and one of them is specified in two files. The rule was already written down; what was missing was a check. Rejected for the same reason human review was rejected as the safety gate |
| A2 | **Write the manifest at the host root, like most tools** | It is the convention, and it is what this ADR refuses. The whole adoption argument is that a maintainer with no time can install this and remove it without archaeology. One directory in, one directory out |
| A3 | **Let `6.5` write README badges directly, as a convenience** | It is a convenience the tool has no standing to take. Contradicts `launch`'s stated posture, and the failure mode — an unexpected diff in a maintainer's front page — costs more trust than a badge earns |
| A4 | **Allow modifying pre-existing files with a `--force` flag** | A flag makes the invariant conditional, and a conditional invariant cannot be gated at 100%. Proposing a diff already covers the legitimate case, and it leaves the decision where it belongs |
| A5 | **Ship a wider allowlist now, for future features** | An allowlist justified by hypothetical need is not closed. Widening it later costs one ADR, which is the correct price for expanding a blast radius |

## Consequences

**Easier.** The install is auditable by listing a directory, and the uninstall is one command.
"Will this tool touch my repo?" has a checkable answer rather than a reassuring one. The gate makes
the claim survive contributors who never read this ADR.

**Harder.** Features that would naturally write to host files — badges, topics, a README section —
must be built as proposal-plus-diff, which is more work than writing the file. Every new write path
needs an allowlist entry and therefore an ADR. The item-1.10 fixture must be a *populated*
repository, not an empty one, or it cannot detect a modification to a pre-existing file.

**Follow-up.** Item 2.2 relocates the manifest; item 6.5 is rescoped to propose-and-diff; item 1.10
becomes the allowlist gate; a non-invasiveness NFR joins the manifest with its measurement stated.
`.specs/` is corrected at the two places that specify the old path — it is a living procedure
document, unlike an ADR, which records a decision at a date and is superseded rather than edited.
