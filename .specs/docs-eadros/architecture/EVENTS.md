# EADROS Event Specification

The event bus is **in-process**, and every event is persisted to the same SQLite file **in the same
transaction as the state change it accompanies**
([ADR-0016](../adr/ADR-0016-local-first-single-file-store.md)). An audit trail committed separately
from the state it describes can disagree with it, and eventually does.

Events follow **CloudEvents 1.0**. [ADR-0002](../adr/ADR-0002-event-driven-architecture.md) requires
schema management without specifying any; this document is that specification.

## Envelope conventions

```json
{
  "specversion": "1.0",
  "type": "dev.eadros.git.commit_pushed.v1",
  "source": "https://github.com/my-org/my-project",
  "subject": "a1b2c3d",
  "id": "01J8F2K9YQ4M7X3N0P5R6T8W2A",
  "time": "2026-07-29T09:14:22Z",
  "datacontenttype": "application/json",
  "correlationid": "01J8F2K9YQ4M7X3N0P5R6T8W2A",
  "causationid": null,
  "taint": "trusted",
  "data": { }
}
```

| Attribute | Rule |
|---|---|
| `type` | **`dev.eadros.<domain>.<event>.v<N>`** — reverse-DNS, with the version *in the type*. Not the folder name: an earlier draft emitted `docs-eadros.git.commit_pushed`, which leaked a documentation directory into the wire contract |
| `source` | A URI, not a path. The repository or subsystem URL |
| `id` | **ULID**, and the **deduplication key**. Handlers must be idempotent: an event delivered twice must produce one effect |
| `time` | ISO-8601 UTC. Required — an event without a time cannot be ordered after a restart |
| `subject` | The thing acted on: a SHA, a post id, a campaign id |
| `correlationid` | Constant across one story's entire journey, from mining to metrics. This is what makes an audit trail readable |
| `causationid` | The `id` of the event that directly caused this one. Correlation groups; causation orders |
| `taint` | **`trusted` \| `untrusted`** — see the trust boundary below |

### Versioning

The version lives in `type`, so a consumer's subscription is explicit about what it accepts.
**Additive, optional fields do not bump the version.** Removing a field, renaming one, or changing a
meaning bumps to `.v2`, and both versions are emitted through a stated deprecation window. A
consumer that receives an unknown `type` **logs and skips**; it never guesses at a shape.

### The trust boundary

`taint: untrusted` marks an event whose payload contains text authored by someone outside the
maintainer set — a commit message on a fork, an external PR title, issue text. Anyone can open a
pull request on a public repository, and its title flows into a pipeline holding publishing
credentials.

**An agent with credentials never receives untrusted content as instruction.** Untrusted payloads
are passed as delimited data, and the `taint` flag propagates through every derived event: a draft
generated from an untrusted candidate is itself untrusted, and the gate's `taint` stage is stricter
on it. This is the envelope-level half of
[ADR-0014](../adr/ADR-0014-deterministic-pre-publish-gate.md); the gate is the other half.

---

## Ingress events

Emitted by observers watching the repository. All may carry `taint: untrusted`.

| Type | Emitted when | Key `data` |
|---|---|---|
| `dev.eadros.git.commit_pushed.v1` | A commit lands on a watched branch | `sha`, `author`, `author_is_maintainer`, `message`, `files_changed`, `insertions`, `deletions` |
| `dev.eadros.git.pr_merged.v1` | A pull request merges | `number`, `title`, `author`, `author_is_first_time`, `commits`, `labels` |
| `dev.eadros.git.release_published.v1` | A release is published | `tag`, `name`, `body`, `prerelease`, `previous_tag` |
| `dev.eadros.ci.gate_intercepted.v1` | A CI gate blocked a change | `pr_number`, `rule`, `rule_ref`, `what_was_prevented`, `log_ref` |
| `dev.eadros.git.issue_closed.v1` | An issue closes with a linked fix | `number`, `title`, `closing_sha`, `labels` |
| `dev.eadros.bench.result_recorded.v1` | A benchmark run completes | `run_id`, `metric`, `value`, `unit`, `baseline_value`, `delta_pct` |

```json
{
  "specversion": "1.0",
  "type": "dev.eadros.ci.gate_intercepted.v1",
  "source": "https://github.com/my-org/my-project",
  "subject": "pr-142",
  "id": "01J8F2M4B7C1D9E2F3G4H5J6K7",
  "time": "2026-07-29T09:20:41Z",
  "datacontenttype": "application/json",
  "correlationid": "01J8F2M4B7C1D9E2F3G4H5J6K7",
  "taint": "untrusted",
  "data": {
    "pr_number": 142,
    "rule": "ADR-0003-agent-orchestration",
    "rule_ref": "docs/adr/ADR-0003.md",
    "what_was_prevented": "dependency injection bypassing the domain boundary",
    "log_ref": "ci/run/8814"
  }
}
```

`taint: untrusted` here because the PR title and body originate outside the maintainer set — and
this is exactly the event class the story miner ranks highest, so the highest-value input is also
the most attacker-reachable one. That coincidence is the reason the boundary exists.

## Internal pipeline events

The events the orchestrator actually runs on. The previous specification defined only two ingress
events and none of these, which left the pipeline's own transitions unobservable — and an
event-driven architecture whose internal steps emit nothing is event-driven in name only.

| Type | Emitted when | Key `data` |
|---|---|---|
| `dev.eadros.mine.candidate_scored.v1` | The miner scores a candidate | `candidate_id`, `score`, `signals`, `archetypes`, `dedup_verdict`, `excluded_reason` |
| `dev.eadros.campaign.opened.v1` | A candidate is promoted | `campaign_id`, `candidate_id`, `archetype` |
| `dev.eadros.draft.created.v1` | A draft is generated | `draft_id`, `channel`, `locale`, `iteration`, `model_tier`, `model_id`, `prompt_version`, `cost_micros` |
| `dev.eadros.gate.evaluated.v1` | A gate pass completes | `subject_id`, `pass`, `stages[]`, `verdict`, `blocking_stage` |
| `dev.eadros.review.requested.v1` | A draft enters the queue | `draft_id`, `surface`, `expires_at` |
| `dev.eadros.review.approved.v1` | A human approves | `approval_id`, `approver`, `mode`, `edited` |
| `dev.eadros.review.rejected.v1` | A human rejects | `draft_id`, `rejected_by`, `reason` |
| `dev.eadros.publish.requested.v1` | Dispatch is queued | `post_id`, `channel`, `tier`, `idempotency_key` |
| `dev.eadros.publish.handoff_required.v1` | `draft` tier reached dispatch | `post_id`, `channel`, `composer_url`, `checklist[]` |
| `dev.eadros.publish.succeeded.v1` | A post goes live | `post_id`, `external_url`, `tracking_url` |
| `dev.eadros.publish.failed.v1` | A dispatch fails | `post_id`, `attempt`, `error`, `next_attempt_at`, `dead_lettered` |
| `dev.eadros.post.retracted.v1` | Content is withdrawn | `post_id`, `class`, `action_taken`, `removal_possible`, `gate_verdict_id` |
| `dev.eadros.metrics.captured.v1` | A snapshot lands | `date`, `posts_covered`, `gaps[]` |
| `dev.eadros.budget.exceeded.v1` | A ceiling is hit | `period`, `ceiling`, `spent`, `action` |

Four of these carry weight beyond their payload:

- **`gate.evaluated`** records both passes and every stage, including the ones that passed. This is
  the data `/eadros audit` reads to compute the block rate **and the false-positive rate** — a gate
  nobody measures is a gate on its way to being switched off.
- **`publish.handoff_required`** is what makes a `draft`-tier channel a first-class citizen rather
  than a gap: the hand-off is an event the system observes, so the Hacker News submission is tracked
  through the same ledger as an automated one.
- **`post.retracted`** carries `gate_verdict_id` — the verdict that *passed* the content that had to
  be withdrawn. That link is the only mechanism by which the gate gets better from being wrong.
- **`budget.exceeded`** is emitted before the action, so a `pause` is explicable after the fact
  rather than mysterious.

## Correlation in practice

One story's whole life shares a `correlationid`, and `causationid` chains it:

```
gate_intercepted ──► candidate_scored ──► campaign_opened ──► draft_created (×3 channels)
                                                                    │
                                          gate_evaluated ◄──────────┘
                                                 │
                                          review_requested ──► review_approved
                                                                    │
                                          publish_requested ◄───────┘
                                            │            │
                              publish_succeeded    handoff_required ──► publish_succeeded
                                            │
                                    metrics_captured (daily, thereafter)
```

Querying `events WHERE correlation_id = ?` returns the complete, ordered account of how a CI failure
became three posts — who approved it, what the gate checked, what it cost, and what it earned.
That query **is** the audit trail ADR-0003 claims as its main benefit, and until these internal
events existed there was nothing to run it against.

## Delivery guarantees

- **At-least-once**, deduplicated on `id`. Handlers must be idempotent; the ones with external
  effects are additionally protected by the outbox and the unique idempotency key
  ([DATA_MODEL](DATA_MODEL.md)).
- **Ordering is per-correlation, not global.** Two campaigns interleave freely; within one,
  `causationid` establishes the order.
- **Replay is supported and read-only.** Replaying the log rebuilds projections; it never re-triggers
  dispatch, because dispatch is gated on outbox state rather than on event arrival. A log you cannot
  safely replay is a log you cannot safely debug.
