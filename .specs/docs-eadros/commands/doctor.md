# `/eadros doctor` — preflight and health

Checks that the environment, credentials and program state are sound. Reports; **fixes nothing**.

Most of what this catches shares a property: it fails **silently and later**, usually mid-campaign.
A token that expired yesterday, a channel policy nobody re-read since March, a metrics snapshot that
has not run in nine days. None of these produce an error until they produce a bad one.

## Checks

### Environment
| Check | Failure means |
|---|---|
| Runtime version | The CLI will not start |
| `git`, and the repository is one | Mining has no input |
| `gh` present and authenticated | Traffic metrics and release events are unavailable |
| Store reachable, WAL enabled, not locked | Another process holds the single writer — `status` names it |

### Credentials
| Check | Why |
|---|---|
| Every enabled channel's credential resolves at `credential_ref` | A missing token is a channel that will fail at dispatch |
| Scopes cover what the adapter needs | A retry cannot fix an insufficient scope |
| **Expiry ≥ 14 days away** | LinkedIn's 60-day credential is the standing case: warn *before* it dies, not after |
| `app_status` is `approved`, not `pending` | A pending app recorded as live silently drops half a weekly cadence |

Values are never read or printed — only that the reference resolves, and what its metadata says.

### Channel policies
| Check | Threshold |
|---|---|
| `policy.verified_on` recency | Warn at **90 days**, block onboarding past it |
| Tier vs current adapter behaviour | A profile claiming `auto` with no write path is a defect |

Platform terms change without telling you. A stale tier assignment is how an account gets banned by a
rule added last quarter, which is why staleness is a finding rather than a note.

### Program health
| Check | Threshold |
|---|---|
| **Metrics snapshot recency** | **Warn at 2 days, critical at 7** |
| Outbox: in-flight older than an hour, or dead-lettered | Needs `publish --reconcile` |
| Review queue: items near expiry | The cadence may exceed what the maintainer can review |
| Budget consumed this period | Against `budget.cost_per_week` |
| Manifest validates, provenance block complete | An incomplete interview block means an unconfirmed assumption |

**The snapshot check is the one with a deadline.** The GitHub Traffic API retains roughly 14 days; a
gap is not late data, it is data that no longer exists
([ADR-0015](../adr/ADR-0015-attribution-methodology.md)). Every other finding here can be fixed
whenever. This one has a clock.

## Output

Three severities — `ok`, `warn`, `critical` — and for anything not `ok`, **the specific action that
resolves it**. A health check that reports a problem without naming its fix has moved the work rather
than reduced it.

Exit non-zero on any `critical`, so it can gate a scheduled run.

## Boundary

Read-only. Rotates nothing, refreshes nothing, publishes nothing. It does not read credential values,
only their metadata.

## Flags

| Flag | Effect |
|---|---|
| `--section <env\|credentials\|policies\|health>` | One section |
| `--json` | Machine-readable, for a cron wrapper |
| `--quiet` | Only `warn` and `critical` |
