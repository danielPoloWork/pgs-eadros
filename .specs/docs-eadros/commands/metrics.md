# `/eadros metrics` — the daily snapshot

Captures post and repository metrics into the append-only time series. Runs **daily**, from cron.

**This is the one job whose failure destroys data rather than delaying it.** The GitHub Traffic API
retains roughly 14 days; a day not captured does not arrive late, it stops existing
([ADR-0015](../adr/ADR-0015-attribution-methodology.md)). Every other command here can be run
whenever. This one has a clock, and that is why it is infrastructure rather than a feature.

## Procedure

1. **Repository traffic** — views, unique visitors, clones, unique cloners, referrers. Written to
   `repo_traffic`, keyed `(repo_id, metric_date)`.
2. **Per-post metrics** — platform engagement where an API exposes it, plus clicks from the owned
   redirect. Written to `post_metrics`, keyed `(post_id, metric_date)`.
3. **Backfill within the window.** Any missing day still inside the API's retention is fetched.
   Outside it, **write a gap row** — `is_gap = 1`, metrics NULL.
4. **Emit** `dev.eadros.metrics.captured.v1` with the days covered and the gaps recorded.

## Gaps are recorded, never inferred

A gap row exists so that an absent measurement can never render as a zero. **Nothing is
interpolated** — a plausible number invented to fill a hole is indistinguishable from a measured one
the moment it is written, and every downstream report would inherit the fiction.

`is_gap` propagates: `digest` names the missing days, `audit` tracks the gap rate against its 1%
threshold, and any lift calculation spanning a gap states so rather than quietly averaging across it.

## Attribution boundaries

Two tables, deliberately not joinable. `repo_traffic` is repository-level and **cannot** be
attributed to a post; `post_metrics` is per post and exact only for the owned-redirect clicks.
Merging them would invite precisely the causal claim ADR-0015 forbids, so the schema does not permit
it ([DATA_MODEL](../architecture/DATA_MODEL.md)).

For a channel whose profile sets `attribution.referrer_preserved: false`, the redirect is the only
per-post signal that survives — recorded as such, not supplemented with a guess.

## Idempotency

Re-running for a date already captured is a no-op, not a duplicate: the primary key refuses it. Safe
to run twice, safe to run after a failure, safe to run from an overlapping cron.

## Boundary

Reads platform APIs and writes only its own two tables. Publishes nothing, drafts nothing, and never
modifies a post.

## Flags

| Flag | Effect |
|---|---|
| `--date <YYYY-MM-DD>` | Capture one specific date within the retention window |
| `--backfill` | Fetch every missing day still inside the window |
| `--json` | Machine-readable |
