# DevRel Intake Interview Protocol

The script the **DevRel Architect** runs to gather everything needed to operate a governed
DevRel program for a repository. Work the phases in order. The machine-readable question bank
is [`questionnaire.yaml`](questionnaire.yaml); this document is the human-readable protocol with
the *why* and the follow-up logic.

This protocol is the sibling of the EADOS intake interview and follows its discipline
deliberately: same provenance vocabulary, same ask-vs-default rule, same "end on the manifest"
closing. Where a rule is inherited verbatim it is cited rather than restated.

## How to run it

- **Import the EADOS manifest first — never re-ask what it already answers.** If the target
  repository was generated or adopted by EADOS, `orchestrator/project.yaml` already holds
  `identity`, `ownership`, `i18n`, `spec.objective`, and — critically — the `announce:` block from
  EADOS **Q4.8** (`channels[]`, each `{name, handle, mode}`). Ingest all of it, mark every carried
  value `imported`, and open Phase 2 from the channels EADOS already recorded. EADROS is the
  deepening of Q4.8, not a second intake. A maintainer answering the same question twice is a bug
  in this protocol.
- **Load the customization overlay.** Read `../config/defaults.yaml`: any value set there overrides
  the built-in default below (precedence: overlay → profile → built-in). Copy a non-empty
  `../config/house-rules.md` body into the manifest's `governance.house_rules`.
- **Check the platform policy dates.** Every [channel profile](channels/_schema.md) carries a
  `policy.verified_on`. **If any channel the maintainer names has a policy older than 90 days, say
  so before Phase 2 settles** — platform terms change under you, and a stale tier assignment is how
  an account gets banned. Re-verification is a normal step, not an error.
- **Conduct it in the maintainer's language.** The answers are transcribed into `devrel.yaml` in
  English (same rule as EADOS `AGENTS.md` §2).
- **Fork first: new program or existing presence?** This protocol frames a **new** DevRel program.
  A repository that already posts — an existing blog, a live X account, past Show HN attempts —
  takes the brownfield intake instead, [`../commands/adopt.md`](../commands/adopt.md): a read-only
  presence map (what exists, what performed, what is dormant), the goal menu, and an `adoption:`
  block in the manifest. It inherits every rule on this list by reference.
- **Ask only what you cannot safely default.** Each question below carries a default. If the
  default is obviously right for the stated project, state the assumption and move on rather than
  asking. Reserve real questions for decisions that change the output.
- **Record provenance as you go — never retrospectively.** The manifest's `interview:` block
  records, for every top-level answer key, whether it was `asked`, `defaulted`, `imported`, or —
  the one value this domain adds — **`inferred_from_sample`** (Phase 3 voice extraction), plus
  `questionnaire_version`. Write each entry the moment the phase settles it; a block reconstructed
  at the end is a guess about your own behavior.
- **Batch related questions.** Phases 0–2 are usually 1–2 rounds. Phase 3 (voice) is the
  substantive conversation, and it ends in a calibration loop, not a questionnaire.
- **Echo back.** After each phase, restate what you captured in one line so the maintainer can
  correct you cheaply.
- **End on the manifest.** The interview's only output is a filled
  [`devrel.yaml`](devrel.yaml.template); you show it for confirmation before anything runs.

---

## Phase 0 — Frame the program

Establish what is being promoted and to whom, so every later question has context.

- **Q0.1 — Which repository, and what does it do in one sentence?** → `REPO`, `PROJECT_TAGLINE`.
  Import from EADOS `identity.project_tagline` when present.
- **Q0.2 — Who should find this?** The reader you are writing for — *"backend engineers evaluating
  rate limiters"*, *"maintainers of agentic tooling"*. Not a demographic; a person with a problem.
  This is the single most load-bearing answer in the interview: it is what the Angle Agent aims at
  and what the Reviewer Agent checks a draft against. → `AUDIENCE`.
- **Q0.3 — What is the goal of the program?** `adoption` (users installing it) / `contribution`
  (external PRs) / `credibility` (the maintainer's own standing) / `hiring` (the repo as a
  portfolio artifact). Default `adoption`. Different goals rank stories differently — a hiring goal
  values the architecture essay that an adoption goal would rank below a how-to. → `PROGRAM_GOAL`.
- **Q0.4 — Repository visibility, now and intended?** `public` / `private` / `private-now-public-later`.
  Drives the confidentiality gate's default posture (Phase 4). A private repo defaults to
  **deny-all** on public channels until an allowlist is set. → `REPO_VISIBILITY`.
- **Q0.5 — Program posture?** `standard` (default) / `enterprise`. Orthogonal to everything else:
  an `enterprise` posture requires legal/comms sign-off in the approval chain, forbids `auto` tier
  on every channel regardless of profile, and makes the disclosure policy mandatory rather than
  optional. Mirrors EADOS `governance.posture` (ADR-0015). → `governance.posture`.

## Phase 1 — Positioning (Wave 0)

**This phase runs before channels, and that ordering is doctrine, not convenience.** Distributing
a project whose README does not explain itself converts attention into bounce — you spend the one
launch you get. `/eadros launch` re-runs this as an audit; here we capture the intent it audits
against.

- **Q1.1 — What is the category, in the reader's existing vocabulary?** The words someone would
  search *before knowing your project exists*. → `CATEGORY`, `KEYWORDS`.
- **Q1.2 — Which projects will a reader compare this to?** Name them. This becomes the competitive
  frame `/eadros launch` scores against, and it lives in the manifest because a hardcoded
  competitor list rots within a quarter. → `COMPARABLES`.
- **Q1.3 — What is the honest differentiator?** One sentence, and it must survive the question
  *"could a competitor's README say this too?"* If it could, it is not a differentiator.
  → `DIFFERENTIATOR`.
- **Q1.4 — What is the proof?** A benchmark, a demo, a diff, a design decision a reader can check.
  Content without proof is the marketing this project exists to avoid. → `PROOF_ASSETS`.
- **Q1.5 — Is there a demo, and where does it live?** A 60–90 s terminal GIF or video, no intro, no
  music, no logo, above the fold. Default: none exists → this becomes roadmap item 1 of the
  program, not a blocker. → `DEMO_REF`.

> **Follow-up trigger (Phase 1):** if Q1.3 comes back as a feature list rather than a
> differentiator, do not proceed to Phase 2. Positioning that is not settled produces content that
> cannot be reviewed, because there is no standard to review it against. Say this plainly and stay
> in Phase 1.

## Phase 2 — Channels

Where content goes, and — separately — **how much autonomy each destination legally permits**.

- **Q2.1 — Which channels do you want to reach?** Offer the profiled set from
  [`channels/`](channels/_schema.md). Import any channel EADOS Q4.8 already recorded and mark it
  `imported`.
  **There is no unsupported channel, only a channel not yet profiled** — if the maintainer names
  one with no profile, that is the normal path: author `channels/<name>.yaml` from
  [`channels/_template.yaml`](channels/_template.yaml), record the ToS verification, add an ADR, then
  resume. Never hardcode a platform's rules into an adapter to skip this.
  → `CHANNELS`.
- **Q2.2 — Confirm each channel's tier.** **The maintainer chooses *whether* to use a channel; the
  channel profile decides *how*.** The tier is read from `channels/<name>.yaml`, not offered as an
  option:
  - **`auto`** — the adapter publishes on approval. (Dev.to, Hashnode, Mastodon, Discord, GitHub.)
  - **`assisted`** — a real write API exists but under quota, app review, or expiring credentials;
    publishing is metered and the budget is a gate. (X, LinkedIn where approved.)
  - **`draft`** — **no lawful automation path.** EADROS prepares the payload, the title, the timing
    and a pre-filled composer link; a human posts it and pastes the URL back so analytics can
    still close the loop. (Hacker News, Reddit, LinkedIn personal profiles.)

  State the tier and the one-line reason from the profile. A maintainer who wants to override a
  `draft` tier upward is asking to risk a **domain-level ban that outlives the account** — refuse
  the silent override and require a recorded ADR with their explicit acceptance. Their project,
  their call; but it is a decision with a paper trail, not a config flag. → `channels[].tier`
  (derived, never asked).
- **Q2.3 — Credentials: where does each token live, and who owns it?** Never the value — the
  *location* (OS keychain entry, secret manager path, env var name) and the human owner. Record
  scope and expiry so `/eadros doctor` can warn before a token dies mid-campaign, not after.
  → `channels[].credential_ref`, `channels[].owner`.
- **Q2.4 — Per-channel cadence ceiling.** Posts per week and minimum gap between posts, defaulted
  from the profile. Developer channels punish frequency far faster than they reward it; this is a
  ceiling the Planner cannot exceed, not a target it tries to hit. → `channels[].cadence`.
- **Q2.5 — Quiet hours and timezone.** No publishing outside a stated window. Default: the
  maintainer's timezone, 08:00–20:00, weekdays. → `SCHEDULE_WINDOW`, `TIMEZONE`.
- **Q2.6 — Per-channel locale.** Which language each channel publishes in. Defaults from the EADOS
  `i18n` block when imported. This is where an "IT + EN LinkedIn" cadence becomes configuration
  instead of a hardcoded assumption. → `channels[].locales`.

> **Follow-up triggers (Phase 2):** any `draft`-tier channel selected → ask who performs the manual
> post and confirm they accept the hand-off loop (Q2.3 owner). Any `assisted`-tier channel → ask
> for the monthly write quota and whether the app is already approved; an unapproved LinkedIn app
> is a **roadmap item, not a channel**, and must not be recorded as live. Locales named beyond the
> canonical one → Phase 3 Q3.7 (translate vs author) becomes mandatory.

## Phase 3 — Voice & narrative

The substantive conversation. **Voice is elicited by sample, not by adjective** — this is the
phase's governing rule and the reason it is not a multiple-choice form.

A maintainer who answers *"professional but approachable"* has told you nothing a model can act on;
every LLM already believes it writes that way, and the result is the generic register that makes
developers reject generated content on sight. Samples carry the information adjectives claim to.

- **Q3.1 — Give me two or three things you have written that sound like you.** Past posts, a README
  you like, an ADR, a long commit message, a comment on an issue you cared about. From these,
  extract a **voice fingerprint** into `voices/<slug>.yaml`
  ([schema](voices/_schema.md)): mean sentence length and variance, person (`I` vs `we` vs
  impersonal), jargon density, hedging frequency, code-to-prose ratio, formatting habits (bullets,
  headers, em-dashes, emoji), and the opening move they actually use.
  Record these as provenance **`inferred_from_sample`**, and — this matters — **show the fingerprint
  back to the maintainer in plain language** before adopting it. An inferred value the maintainer
  never saw is indistinguishable from a guess.
  If no sample exists, fall back to a seeded profile from [`voices/`](voices/_schema.md) and say
  which, explicitly. → `VOICE_PROFILE`, `VOICE_SAMPLES`.
- **Q3.2 — Person and byline.** Does the account speak as **a person** (`I built…`) or as **the
  project** (`v2.0 ships…`)? Mixed bylines read as inauthentic faster than either one alone.
  → `voice.person`, `voice.byline`.
- **Q3.3 — Which story shapes are you willing to publish?** Not a style question — a **consent**
  question, and the answers configure the Angle Agent's admissible set. Offer each archetype and
  capture an opt-in plus a frequency ceiling:

  | Archetype | What it publishes | Default |
  |---|---|---|
  | `postmortem` | a bug you shipped and how it was caught | **ask — never default on** |
  | `subtraction` | code deleted, complexity removed | on |
  | `tradeoff` | an ADR walked through, including the cost accepted | on |
  | `benchmark` | a measured improvement, with method | on, `requires_evidence` |
  | `buildlog` | progress, in-flight work | on |
  | `howto` | teaching the reader something reusable | on |
  | `opinion` | a thesis stated in the maintainer's name | **ask — never default on** |
  | `release` | what shipped and why it matters | on |

  `postmortem` and `opinion` default **off** and must be asked. Publishing a failure story or a
  stated opinion under someone's name without their explicit consent is a reputational act the
  maintainer did not authorize — and a tool that does it once will not be trusted again.
  → `voice.archetypes`.
- **Q3.4 — The forbidden list.** Words, phrases and constructions that must never appear. Seed it
  from the profile and let the maintainer add their own. This is a **deterministic lint**, not an
  LLM judgment call — it is the only reliable part of the "zero marketing buzzwords" rule.
  Seeded defaults include the vocabulary (*revolutionary*, *game-changing*, *seamless*, *leverage*,
  *unlock*, *delve*) **and the constructions**, which matter more: the rhetorical-question opener,
  the "it's not X — it's Y" antithesis, the emoji section header, the "in today's fast-paced world"
  preamble, and the three-item parallel triad that generated prose falls into by default.
  → `voice.forbidden`.
- **Q3.5 — Claim discipline.** Confirm the standing rule and capture any exception: **every factual
  claim must carry a source reference** — a commit SHA, a PR number, a `file:line`, a benchmark run
  id — and **code snippets are extracted verbatim from the repository at a stated SHA, never
  generated.** The Reviewer Agent resolves each reference mechanically; an unresolvable claim fails
  the draft rather than being softened. → `voice.claim_discipline` (default `strict`).
- **Q3.6 — Disclosure.** Is AI assistance disclosed, where, and in what words? Default: a short
  standing line in the maintainer's own phrasing, applied per channel where the channel permits it.
  Mandatory when `governance.posture: enterprise`. → `voice.disclosure`.
- **Q3.7 — Locale strategy (only when Phase 2 named more than one locale).** For each non-canonical
  locale: **`translated`** (derived from the canonical draft — cheap, and correct for release notes
  and factual posts) or **`authored`** (written natively for that audience — necessary for anything
  whose value is voice, because a translated opinion reads as translated). Default `translated`,
  and say so. → `locales[].mode`.

> **Calibration loop — the phase does not end on an answer, it ends on a sample.**
> Before writing the voice profile, take **one real recent commit from the repository** and
> generate three short drafts under the captured profile. Ask the maintainer to pick one and edit
> it freely. **Diff their edit against your draft and fold the delta back into the fingerprint** —
> what they cut is worth more than what they kept. Record the round in `voice.calibration[]`
> (`{date, source_ref, rounds, accepted}`). Two rounds is normal; converging on the first is
> suspicious and usually means the maintainer was being polite. This loop is what makes the profile
> a measurement rather than a self-description, and it is re-runnable at any time via
> `/eadros onboard --recalibrate`.

## Phase 4 — Governance, safety & budget

What may be published, who authorizes it, and what it is allowed to cost.

- **Q4.1 — What is publishable?** A path allowlist of repository areas whose content may be quoted
  or shown (`src/**`, `docs/**`), and a deny-list that outranks it. Default for a **private** repo
  is **deny-all**: nothing is quotable until the maintainer opts paths in. Also capture the maximum
  number of diff lines quotable in one post. → `safety.path_allowlist`, `safety.deny`,
  `safety.max_diff_lines`.
- **Q4.2 — What must never appear?** Customer names, internal hostnames, unreleased feature names,
  employer identifiers. Seeds the deny-list that runs **before** any model sees the content and
  again **before** publish. State plainly that secret scanning runs unconditionally and is not
  configurable off. → `safety.deny_terms`.
- **Q4.3 — Embargo rules.** Default: nothing is publishable from an unmerged branch, from a commit
  younger than N hours (default 24 — it lets a bad commit be reverted before it is famous), or from
  a repository still marked private. → `safety.embargo`.
- **Q4.4 — Who approves, and through which surface?** CLI, a local review dashboard, or a
  Discord/Telegram approval channel. Capture the approver identities. Under
  `posture: enterprise`, a second approver is required for any post carrying a `benchmark` or
  `opinion` archetype. → `governance.approvers`, `governance.review_surface`.
- **Q4.5 — Approval strength.** Default **`edit-required`**: approval is only valid if the final
  text differs from the generated draft, or the approver passes an explicit `--as-is` flag that is
  recorded in the audit trail. This converts the human gate from a rubber stamp into an editorial
  act — the difference between HITL as governance and HITL as decoration. → `governance.approval_mode`.
- **Q4.6 — Budget ceilings.** Numeric, never adjectives — the same discipline as an EADOS hard NFR
  budget, and enforced the same way (a gate, not a promise): **cost per week** in the maintainer's
  currency, **max campaigns mined per run**, **max reviewer↔copywriter iterations** (default 2 — an
  unbounded critique loop is this architecture's classic cost failure), and a **hard stop** action
  when a ceiling is hit (`pause` / `warn`). → `budget.*`.
- **Q4.7 — Who holds the kill switch?** The human who can run `/eadros pause` and the channel they
  will be reached on. Confirm they know they hold it. → `governance.kill_switch_owner`.

## Phase 5 — Cadence & campaign roadmap

- **Q5.1 — Weekly cadence.** What one normal week looks like, within the Phase 2 ceilings. Default
  is deliberately low: **one substantive post per week plus one opportunistic**. Volume is the most
  common way to burn a developer channel, and the ceiling is the honest setting.
  → `cadence.weekly`.
- **Q5.2 — Milestone campaigns, defined up front.** Lay out all of them now, not one at a time —
  v1.0, a Show HN, an Awesome-list submission, a conference talk. For each: `trigger`, `channels`,
  `archetype`, and the **pre-condition that must hold before it may fire** (e.g. *"Show HN only
  after the Wave 0 audit scores ≥ 80 and the demo exists"*). A campaign with no pre-condition is
  how a project spends its one launch on a README that was not ready. → `campaigns[]`.
- **Q5.3 — What does success look like in 90 days?** One primary metric with a number, chosen from
  `SUCCESS_METRICS.md`, plus the honest note that attribution is directional, never causal — see
  the attribution-methodology ADR (proposed, not yet written). A KPI without a target is a poster.
  → `success.primary_metric`, `success.target`.

---

## Closing the interview

1. Merge the answers with the resolved channel and voice profiles into
   `orchestrator/devrel.yaml`, and verify the `interview:` provenance block you filled as you went
   is complete — one entry per top-level answer key, `questionnaire_version` set.
2. **Present the manifest**, pointing the maintainer at every entry marked **`defaulted`** and
   every entry marked **`inferred_from_sample`**. Those are the two classes of value you assumed
   rather than were told; a considered answer and a silent assumption must never look the same at
   confirmation time.
3. **Present the tier table separately and explicitly** — which channels will publish on approval,
   which are metered, and which require a human to press the button. A maintainer who is surprised
   later by a `draft`-tier hand-off was not properly onboarded here.
4. On confirmation, the program is live: `/eadros status` reports it and `/eadros mine` can run.
   On any change, edit the manifest and re-confirm — never operate from an unconfirmed manifest.
