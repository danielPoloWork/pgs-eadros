# Eval suite: `channels` — formatter conformance and platform contracts

Two deterministic suites over the publishing edge: does the text we generate fit the destination,
and do we handle what the destination sends back?

## Formatter snapshots

Cheap, exact, and the fastest way to catch a regression that would otherwise be discovered by
readers.

For every channel profile, snapshot the rendered output of a fixed draft set and assert the
profile's constraints hold:

| Assertion | Source |
|---|---|
| Length within `limits.max_chars` | profile |
| Markdown flavour respected — no fenced code where `markdown: none` | profile |
| Code rendered per `limits.code_blocks` (`fenced` / `image-only` / `unsupported`) | profile |
| Tag count and style within `limits.tags` | profile |
| `canonical_url` set to the project's own docs where `format.canonical_url: supported` | profile |
| Thread emitted only where `format.thread: supported` | profile |
| Tracking URL present, except where `attribution.utm: unsupported` | [ADR-0015](../adr/ADR-0015-attribution-methodology.md) |

Two of these are worth more than they look:

**`canonical_url`** — syndicating without it lets the platform outrank the source you are trying to
grow, which inverts the point of publishing. It is a silent failure with no symptom for months, so
it gets a test rather than a code review.

**LinkedIn's `markdown: none`** — the copywriter must not emit fenced code for a channel that
renders it as literal backticks. This is the class of mistake that makes automated posting
recognisable at a glance.

**Property test, across all profiles:** no formatter output ever exceeds its profile's `max_chars`,
for any input. Generated rather than enumerated — truncation bugs live at the boundary, and the
boundary is where a snapshot set never quite lands.

## Platform contract tests

You cannot exercise LinkedIn from CI, and you should not try. Contracts are verified against
**recorded response fixtures**.

What they verify is **the error taxonomy**, not the happy path — the happy path is the case that
already works:

| Response | Required behaviour |
|---|---|
| `2xx` | Outbox outcome committed with `external_url`; state → `published` |
| `401` / expired credential | State → `failed`, **not retried**; `/eadros doctor` surfaces the credential |
| `403` insufficient scope | `failed`, with the missing scope named — a retry cannot fix a scope |
| `429` rate limited | Backoff with jitter, honouring `Retry-After`; quota decremented |
| `5xx` | Retry with exponential backoff; dead-letter on exhaustion |
| Timeout, no response | Outcome `unknown` — **never retried blind**; `--reconcile` resolves it |
| Duplicate detected by platform | Reconciled to the existing post, not re-created |

**The timeout row is the one that matters.** It is the exact path by which a naive retry loop
becomes a public double post, and it is the reason the outbox commits intent before the call
([`publish.md`](../commands/publish.md)). The test kills the process between intent and outcome and
asserts `--reconcile` converges on exactly one post — the same property the state-machine suite
asserts from the other side.

### Fixture hygiene

- **Fixtures are scrubbed.** A recorded response carrying a real token committed to a repository is
  an incident, and it is a common one. Scrubbing is asserted by a test over the fixture directory,
  not left to reviewer attention.
- **Fixtures carry `verified_on`**, the same 90-day clock as channel profiles. A fixture is a
  snapshot of an API that has since moved, and a suite that only ever sees stale fixtures ends up
  measuring the fixtures.
- A **nightly job** exercises `auto`-tier channels against live sandboxes where one exists, and
  reports drift between the fixture and reality. Drift is a finding, not a failure — but an
  unreported drift is how a green suite accompanies a broken adapter.

## Tier routing

The smallest suite here and the one whose failure is worst:

1. **A `draft`-tier channel has no dispatch path**, under any input, in any state. Asserted by
   attempting dispatch on every `draft` profile and requiring the attempt to be structurally
   impossible rather than merely refused.
2. **An `assisted` channel refuses when quota is spent** and never borrows from the next period.
3. **A channel with `app_status: pending` is not publishable**, and the attempt is an explicit error
   rather than a silent skip — a weekly cadence that quietly ships half its output is worse than one
   that stops and says so.
4. **`tier_at_publish` is recorded from the profile as applied**, so changing a profile afterwards
   does not rewrite history ([ADR-0011](../adr/ADR-0011-channel-capability-tiers.md)).
