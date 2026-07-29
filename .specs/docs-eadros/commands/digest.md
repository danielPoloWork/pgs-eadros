# `/eadros digest` — the maintainer summary

What shipped, what it cost, what moved. Read-only, and the one report a maintainer actually reads.

## Output

```
digest · 2026-07-22 → 2026-07-29

shipped   3 posts · 2 channels
          "Why the gate caught a boundary violation"   devto/en    · 412 clicks
          "What we deleted and why"                    linkedin/it · 88 clicks
          "Show HN: EADROS"                            hackernews  · you posted

cost      €0.44 · €0.15/post (target €0.15)
          triage €0.02 · drafting €0.19 · review €0.23

your time 14 min in review · 3 approved · 1 rejected · median edit 18%

repo      unique cloners 34 (+9 vs prior 7d)
          unique visitors 210 (+31)
          lift after devto post: +7 cloners over 72 h vs a 14-day baseline
          ── directional, not causal · n=1 · see ADR-0015

not measured
          linkedin  referrer stripped — clicks are from the redirect only
          hackernews  no per-post metrics available
          2026-07-24  snapshot gap — no traffic data exists for this day

blocked   show-hn-v2 · wave-0 score 62 < 80
```

## The three sections that make it honest

**`your time`** is the tool's own success metric, reported to the person paying it. If EADROS costs
more attention than writing the posts by hand would have, every other number here is decoration
([SUCCESS_METRICS](../vision/SUCCESS_METRICS.md)). Median edit percentage sits next to it because,
read against the rejection rate, it is the earliest signal that review has become a stamp.

**Every effect is labelled `directional` with its sample size.** The lift line states its control
window and refuses the word *caused*. At one post per week, per-angle comparison never reaches
statistical power, and a digest implying otherwise would let a maintainer make real content decisions
on noise ([ADR-0015](../adr/ADR-0015-attribution-methodology.md)).

**`not measured` is a required section, never omitted when empty of good news.** It names the channels
where attribution is lost and the days the snapshot missed. **An absent number and a zero must never
render the same** — a stripped referrer reported as "0 clicks" is a lie the system told itself first.

## Cadence

Weekly by default, from cron alongside the `weekly` run. Optionally delivered to
`governance.review_surface`, so it arrives where the maintainer already is.

## Boundary

Read-only. Reads the store; queries no platform (the daily `metrics` collector already did), writes
nothing, publishes nothing.

## Flags

| Flag | Effect |
|---|---|
| `--period <7d\|30d\|90d>` | Window (default 7d) |
| `--since <date>` | Explicit start |
| `--json` | Machine-readable |
