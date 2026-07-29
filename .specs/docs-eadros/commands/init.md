# `/eadros init` — initialize a governed DevRel program

The **entry command** of the pipeline. It frames the program and writes the initial manifest so
every later phase has state to read. Owned by the **devrel-architect** role.

`init` frames a **new** program. A repository that already posts — a live X account, an existing
blog, past Show HN attempts — takes its brownfield sibling [`/eadros adopt`](adopt.md) instead:
presence map → goal menu → a legal route into `onboard`.

## Procedure

1. **Preflight.** Run `/eadros doctor`. It checks the runtime, `git`, `gh` and auth, and reports
   which channel credentials are present. Nothing here requires credentials yet — but a maintainer
   who learns at Phase 2 that they hold no LinkedIn token has answered a page of questions for
   nothing. Surface the gaps now.

2. **Import the upstream manifest — before asking anything.** If `orchestrator/project.yaml`
   exists, the repository is EADOS-governed. Read `identity`, `ownership`, `i18n`, `spec.objective`
   and the `announce:` block (EADOS **Q4.8**), and record every carried value as `imported`.
   Write the source path into `upstream.eados_manifest` and the key list into
   `upstream.imported_keys`.

   **EADROS is the deepening of Q4.8, not a second intake.** A maintainer who already told EADOS
   which channels they announce on must not be asked again; being asked twice is the clearest
   possible signal that two tools were bolted together rather than designed as one.

3. **Frame.** Run interview [Phase 0](../orchestrator/interview.md) (Q0.1–Q0.5) and
   [Phase 1](../orchestrator/interview.md) (Q1.1–Q1.5). Phase 1 is positioning, and it runs here —
   before any channel is chosen — because that ordering is the product's central claim. A program
   that picks channels before it can state its differentiator has already decided to distribute
   something it cannot describe.

   If Q1.3 returns a feature list rather than a differentiator, **stay in Phase 1 and say why.**
   Positioning that is not settled produces content that cannot be reviewed, because there is no
   standard to review it against.

4. **Write the manifest skeleton.** Copy
   [`devrel.yaml.template`](../orchestrator/devrel.yaml.template) to **`.eadros-core/devrel.yaml`**
   — inside the installation root, never at the host project's root, which would create an
   `orchestrator/` directory in a repository EADROS does not own (ADR-0004) —
   and fill `schema_version`, `upstream`, `program`, and `positioning`. Leave `channels`, `voice`,
   `safety`, `governance` and `budget` to `/eadros onboard` — writing a plausible default into a
   safety field the maintainer never saw is exactly the failure this manifest's provenance block
   exists to prevent.

5. **Confirm.** Present the skeleton. **Echo every assumed default explicitly**, glossed
   *assumed* — an answer taken from the questionnaire default (provenance `defaulted`) must never
   look the same as one the maintainer gave. A considered answer and a silent assumption reaching
   confirmation indistinguishable from each other is how a wrong default survives into production.

6. **Report the next move.** `init` always hands off to [`/eadros onboard`](onboard.md), which
   settles channels, voice, safety and budget. State plainly that **no content can be mined or
   drafted until `onboard` completes** — the pipeline has no safe defaults for what may be
   published or what it may cost.

## Boundary

The agent **drafts** the manifest and **proposes** the hand-off; the **human confirms**. `init`
creates no campaign, contacts no platform, spends no tokens beyond the interview itself, and
publishes nothing. It cannot: no channel is configured yet.

## Command surface

Once the manifest exists, offer to generate the host's adapter tree. **Resolve the host explicitly
and say which one you used** — manifest `routing.host`, then the catalog's detection markers, then
ask. Never default silently: a silent default writes a command tree the maintainer's actual host
will never read.

Where a host has no command mechanism, say so — the surface is the documented procedure plus the
CLI, which works everywhere.
