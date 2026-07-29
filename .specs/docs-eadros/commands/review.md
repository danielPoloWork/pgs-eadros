# `/eadros review` · `approve` · `reject` — the human gate

The queue where a person decides. **Human only** — no agent path exists to `approve` or `reject`,
and an implementation that provides one has removed the thing this product sells.

## What the queue must show

A reviewer shown only the draft can contribute taste. A reviewer shown *what the machine already
checked* can contribute judgment — which is the scarce thing, and the reason the strong-tier model
runs before this point rather than after it.

Each item therefore renders:

| Section | Why it is there |
|---|---|
| **The draft**, per channel and locale | The thing being decided |
| **Claims**, each with its resolution status | So the reviewer checks *meaning*, not *existence* — the resolver already did existence |
| **Reviewer objections**, per class with spans | Including the open ones carried over when the iteration cap was hit |
| **Gate verdicts** that passed | What was checked, so the human knows what they are *not* responsible for |
| **Channel, tier, and scheduled time** | A `draft`-tier item means *you* will post it — surprising someone with that later is an onboarding failure |
| **Cost so far** | The one number that makes "reject" feel like a real decision |
| **Expiry** | When this goes stale and is dropped |

Showing the gate verdicts that **passed** is the non-obvious one. It tells the reviewer that secrets,
paths and embargo are handled, so their attention goes to the claim that reads oddly rather than to
re-doing a mechanical check.

## `approve`

```
eadros approve --id <draft-id>
```

Records an `approvals` row: approver, timestamp, mode, `hash_before`, `hash_after`, final body.

**Approval is invalid when the text is unchanged**, unless `--as-is` is passed. This is enforced by
the database, not by the command:

```sql
CHECK (mode = 'as-is' OR hash_before <> hash_after)
```

The policy converts the human gate from a rubber stamp into an editorial act
([ADR-0014](../adr/ADR-0014-deterministic-pre-publish-gate.md)). `--as-is` remains available and
legitimate — a draft can genuinely be right — but it is **recorded with the approver's identity**, so
a program running on `--as-is` is visible in `/eadros audit` rather than indistinguishable from one
where someone is reading.

Approval moves the post to `approved`. It does **not** publish; [`publish`](publish.md) does, under
its own guards.

## `reject`

```
eadros reject --id <draft-id> --reason "..."
```

The reason is required, and it is not bureaucracy: every rejection is a **labelled false negative** —
a draft a person threw out that the Reviewer passed — written back into the eval corpus by
`/eadros eval --add-from-rejections`. It is the most valuable case type available, because nobody had
to invent it ([eval/reviewer](../eval/reviewer.md)).

## Expiry

A queued draft has a staleness window. Past it the post moves to `expired` — **reported, never
silently published**. A post about a commit from three weeks ago is not news, and shipping it late is
worse than not shipping it.

Expiry is also a signal about the program: a queue that regularly expires means the cadence is above
what the maintainer can actually review, and the fix is lowering `cadence.weekly`, not reviewing
faster.

## Surfaces

`governance.review_surface` selects where the queue appears: `cli`, a local dashboard, or a
Discord/Telegram approval channel. The gate must reach the maintainer where they already are — a
queue that requires opening a terminal is a queue that waits until Friday.

Under `governance.posture: enterprise`, a **second approver** is required for any post carrying a
`benchmark` or `opinion` archetype.

## Boundary

`review` is read-only. `approve` and `reject` write approval and rejection records and nothing else —
neither dispatches, neither contacts a platform. **An agent may never invoke either.**

## Flags

| Flag | Effect |
|---|---|
| `--id <id>` | One item instead of the queue |
| `--as-is` | Approve unedited. Recorded with the approver's identity, never silent |
| `--reason "..."` | Required on `reject` |
| `--oldest` | Open the item closest to expiry |
| `--channel <name>` | Filter the queue |
