# `/eadros status` — where everything is

The read-only view of the program. Answers *what is in flight, what is waiting on me, and what is
stuck*.

## Output

```
program   my-org/my-project · goal: adoption · posture: standard
          PAUSED since 2026-07-29 09:14 by Daniel Polo — "leak in p-42"

campaigns drafting 1 · in_review 2 · partially_published 1 · published 14

review    3 waiting · oldest 4 h (expires in 44 h)
          p-88 devto/en · p-89 linkedin/it · p-91 hackernews/en (draft tier — you post)

outbox    1 in flight · 0 dead-lettered
          p-87 linkedin — outcome unknown for 22 min → run publish --reconcile

handoffs  1 awaiting your post
          p-91 hackernews — composer link ready, 5-item checklist

budget    €2.41 / €5.00 this week · 14 posts · €0.17/post (target €0.15)

metrics   last snapshot 14 h ago · 0 gaps in 30 days

next      weekly run Mon 09:00 · show-hn-v1 blocked (wave-0 score 62 < 80)
```

Four lines are doing real work here.

**The pause banner is first**, with who paused it and why. A paused program that looks normal is how
a kill switch gets forgotten for a week.

**`partially_published` is reported as a normal state**, not an error. A campaign live on Dev.to,
awaiting a human on Hacker News and failed on LinkedIn is one row honestly described — that outcome is
common, and flagging it red would train the maintainer to ignore red
([STATE_MACHINE](../architecture/STATE_MACHINE.md)).

**Handoffs are listed separately from the review queue.** They are the maintainer's *action*, not
their *decision* — and burying a `draft`-tier hand-off inside a queue of things to read is how a Show
HN submission sits unposted for three days.

**Anything stuck names its command.** `outcome unknown for 22 min → run publish --reconcile`. A
status view that reports a problem without naming its fix has moved work rather than reduced it.

## Boundary

Read-only. Reconciles nothing, retries nothing, publishes nothing. It reports the lock holder when
another process holds the single writer ([ADR-0016](../adr/ADR-0016-local-first-single-file-store.md)).

## Flags

| Flag | Effect |
|---|---|
| `--campaign <id>` | Full detail for one campaign, all its posts and their transitions |
| `--stuck` | Only items needing action: dead letters, unknown outcomes, expiring reviews |
| `--json` | Machine-readable |
