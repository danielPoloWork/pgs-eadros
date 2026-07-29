# `/eadros weekly` — the cadence run

Mines, ranks and drafts one week's content, within every configured ceiling. Owned by the
**scheduler**.

**Everything about this run comes from the manifest.** The v1 specification hardcoded *"1 substantial
post, 2 LinkedIn posts (IT & EN), 1 Dev.to article, 1 Awesome-list PR, 1 demo update"* — that was one
maintainer's personal workflow written into a product, and it is exactly the kind of assumption that
makes a tool unusable for its second user.

## Procedure

1. **Preflight.** Refuse if paused. Warn on anything `doctor` marks critical — a run that drafts into
   an expired credential wastes the whole week's budget.
2. **Compute headroom**, per channel, before mining. Channel ceiling minus posts already published
   this period, minus what is already queued. **A channel at its ceiling is excluded from mining**,
   so nothing is drafted that could not be published anyway.
3. **Mine.** Deterministic scoring across the window since `pipeline_state.last_mine_at`
   ([mine](mine.md)).
4. **Allocate against `cadence.weekly`.** Default `{substantive: 1, opportunistic: 1}` — deliberately
   low. Volume is the fastest way to burn a developer channel, and the ceiling is the honest setting
   rather than a starting point to grow from.
5. **Draft**, top-ranked first, stopping at the first of: the cadence allocation, the channel
   headroom, or `budget.cost_per_week`.
6. **Queue for review.** Nothing publishes.
7. **Report** what was drafted, what was cut and why, and what it cost.

## The ceilings are ceilings

`cadence.weekly` is not a quota to fill. If mining produces one candidate worth telling, the week
ships one post — **the run does not reach for a second to hit a number.** A DevRel tool that
manufactures content to satisfy its own schedule has become the thing this project exists to replace.

An empty week is a valid, reported outcome:

```
weekly · no candidates above threshold in the last 7 days
  12 commits mined · 12 excluded (9 dependency bumps, 3 formatting)
  0 drafted · €0.00 spent
```

That is a **successful run**. It cost nothing and correctly said there was nothing to say.

## Locales

Per-channel, from `channels[].locales` and `locales[].mode`. A `translated` locale derives from the
canonical draft — one generation plus a translation pass. An `authored` locale is a separate draft
against the same angle, and costs accordingly. The interview captures which, so a bilingual cadence
is a configuration rather than an assumption.

## Scheduling

Invoked from cron or CI, not from a resident daemon — a long-lived process holding publishing
credentials is a standing risk for no gain
([CONTAINERS](../architecture/CONTAINERS.md)). Drafting may run at any hour; only dispatch is bound
by `schedule.window`.

## Boundary

Mines, drafts, queues. Never approves, never publishes. Refuses while paused, and never exceeds a
channel ceiling, the cadence allocation or the weekly budget.

## Flags

| Flag | Effect |
|---|---|
| `--dry-run` | Mine and rank, show what would be drafted, generate nothing |
| `--explain` | Ranking signals and a cost estimate before spending |
| `--since <ref>` | Override the mining window |
