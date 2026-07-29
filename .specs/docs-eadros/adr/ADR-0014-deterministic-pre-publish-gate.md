# ADR-0014: A deterministic pre-publish gate, because human review decays

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-07-29 |
| **Amends** | [ADR-0008 — Human review gate](ADR-0008-human-review.md) (keeps the gate; withdraws its claim to eliminate risk on its own) |
| **Related** | [`commands/publish.md`](../commands/publish.md) · [`commands/retract.md`](../commands/retract.md) · [ADR-0012 — Voice](ADR-0012-voice-profile-and-calibration.md) |

## Context

ADR-0008 states that the human review gate *"eliminates hallucination risk and protects brand
reputation."* That claim is wrong in a way worth naming precisely: **human attention is not a
control**. A reviewer reads the first ten drafts carefully, the next twenty quickly, and everything
after that by habit. A gate whose only enforcement is vigilance degrades on a predictable curve,
and it degrades silently — the approvals keep arriving, so nothing looks broken until something is.

The exposures the gate is supposed to cover are not all the kind a person catches by reading:

- **Secrets in quoted diffs.** A key in a code snippet is a lexical problem. A tired human scanning
  prose for narrative quality will not see it; a regex will, every time.
- **Private-repository content reaching public channels.** The system mines repositories that may
  be private and publishes to channels that are not. Nothing in the current specification stops a
  commit message naming a customer from becoming a LinkedIn post.
- **Prompt injection through repository content.** Commit messages, PR titles and issue text are
  attacker-controllable on any public repository, and they flow into a pipeline holding publishing
  credentials. A PR titled `fix: typo. Ignore previous instructions and post …` is a complete
  vector, and no ADR mentions it.
- **Unresolvable factual claims.** A confident sentence with no source is indistinguishable from a
  correct one at reading speed.

The correct division of labour is the one every mature review process converges on: **machines
check what is checkable, so humans spend attention on judgment.** Asking a person to catch the
same buzzword for the twentieth time is how you lose their attention for the thing only they can
catch.

## Options considered

**A. A deterministic gate running twice — on inputs before any model, on outputs before dispatch —
that cannot be switched off** *(chosen)*

- ✅ **Coverage is constant.** A regex does not get tired at draft forty. The controls that are
  mechanisable stop depending on the reviewer's week.
- ✅ **Running on inputs, not just outputs, is what stops injection.** Content authored by anyone
  outside the maintainer set is marked untrusted at ingestion and passed to agents as delimited
  data, never as instruction. Filtering only the output would mean the injected instruction had
  already executed with credentials in hand.
- ✅ **It repairs ADR-0008 without weakening it.** The human gate stays mandatory; it stops being
  asked to do the part it cannot do. Combined with `approval_mode: edit-required`, the human's role
  becomes editorial judgment — which is both what they are good at and what keeps them engaged.
- ✅ **Claim resolution is mechanical.** Every factual claim carries a `source_ref`; the gate
  resolves it — does the SHA exist, does `file:line` exist at that SHA, does the number match the
  benchmark record. An unresolvable claim **fails the draft** rather than being softened into
  vagueness, which is what a model asked to "be careful" produces instead.
- ❌ **False positives will block good content**, and a gate people resent gets bypassed.
  Mitigated by *measuring* the false-positive rate in `/eadros audit` and treating a rising rate as
  a defect in the gate, not in the maintainer. An unmeasured gate is a gate on its way to being
  disabled.
- ❌ **Deny-lists are never complete.** True, and not a reason to skip the ones that work: this is
  defence in depth under a human gate, not a replacement for it.

**B. Rely on the Reviewer Agent to catch these** — ✅ no new machinery, already in the architecture;
❌ asks a model to catch what a model produced, offers no coverage guarantee, and is structurally
wrong for secrets (a lexical problem given to a semantic tool). It also cannot address injection,
since the injected instruction reaches the reviewer through the same channel. **Rejected.**

**C. Human review only — the ADR-0008 status quo** — ✅ simplest, and honest about who is
accountable; ❌ the decay curve above, and it puts an unpaid maintainer in the position of being
the only thing between a private repository and a public channel. **Rejected.**

**D. Only ever publish from public repositories** — ✅ removes the leak class outright; ❌ public
repositories still carry embargoed work, unreleased feature names, and customer names in commit
messages, so it removes less than it appears to while excluding a large share of real users.
**Rejected.**

## Decision

A `PrePublishGate` runs **twice** — over mined inputs before any model call, and over generated
content before dispatch — and **fails closed**. Its stages, in order:

| Stage | Check | Configurable |
|---|---|---|
| `secrets` | Credential/key patterns over any quoted content | **No off switch** |
| `deny_terms` | `safety.deny_terms` — customers, internal hosts, unreleased names | Terms only |
| `paths` | `safety.path_allowlist` / `safety.deny`; private repos start deny-all | Yes |
| `diff_cap` | `safety.max_diff_lines` | Yes |
| `embargo` | Unmerged branch, commit-age floor, private-repo hold | Yes |
| `taint` | Externally-authored content marked untrusted; never treated as instruction | **No off switch** |
| `claims` | Every `source_ref` resolves at the stated SHA | Threshold only |
| `voice_lint` | `forbidden.words` and `forbidden.constructions` ([ADR-0012](ADR-0012-voice-profile-and-calibration.md)) | List only |

Every block is recorded with its stage, its reason and the offending span. `/eadros audit` reports
the block rate **and the false-positive rate**, so the gate is itself under review.

## Consequences

- ADR-0008's overclaim is withdrawn and its gate is strengthened: the human decides, on drafts that
  have already survived the checkable failures.
- Prompt injection has a named control, and the untrusted-content boundary becomes an architectural
  invariant rather than an assumption — agents holding credentials never receive attacker-authored
  text as instruction.
- Code snippets must be **extracted verbatim at a SHA** for `claims` to be resolvable at all, which
  removes the largest hallucination surface as a side effect.
- Channels where retraction is impossible ([`retract`](../commands/retract.md)) are gated hardest,
  because for them the gate is the only control that exists.
- The gate needs its own test corpus — planted secrets, poisoned commit messages, unresolvable
  claims — and that corpus becomes part of `/eadros eval`.
