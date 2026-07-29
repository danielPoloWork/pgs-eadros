# EADROS Sequence Diagrams

Both diagrams are the runtime view of [`STATE_MACHINE.md`](STATE_MACHINE.md); where they disagree,
the state machine is authoritative.

## 1. Gate intercept → published, across two tiers

The normal path, and the one that shows why the tier model exists: the same approved story reaches
one channel automatically and another through a human.

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Maintainer
    participant CI as CI gate
    participant Bus as Event bus
    participant Miner as Story miner (no model)
    participant Pipe as Angle → Copywriter → Reviewer
    participant Gate as PrePublishGate (deterministic)
    participant Queue as Review queue
    participant Pub as Publisher + outbox
    participant Auto as Dev.to (tier auto)
    participant Draft as Hacker News (tier draft)

    CI->>Bus: ci.gate_intercepted.v1 (taint: untrusted)
    Bus->>Gate: input pass — secrets, paths, embargo, taint
    Gate-->>Bus: pass
    Bus->>Miner: score candidate
    Miner-->>Bus: mine.candidate_scored.v1 (score, signals, archetypes)
    Note over Miner: deterministic, no model call —<br/>only the top K reach the pipeline
    Bus->>Pipe: campaign.opened.v1
    Pipe->>Pipe: reviewer loop (max 2 iterations)
    Pipe-->>Gate: draft.created.v1 (×2 channels)
    Gate-->>Queue: gate.evaluated.v1 — output pass
    Queue->>Dev: review.requested.v1
    Dev->>Queue: edits, then approves
    Note over Dev,Queue: approval invalid if text is unchanged,<br/>unless --as-is, which is recorded
    Queue-->>Pub: review.approved.v1
    Pub->>Pub: reserve idempotency key + commit outbox intent
    Pub->>Auto: dispatch
    Auto-->>Pub: 201 + URL
    Pub-->>Bus: publish.succeeded.v1
    Pub->>Dev: publish.handoff_required.v1 — payload, composer link, checklist
    Note over Pub,Draft: draft tier has NO dispatch path
    Dev->>Draft: submits manually
    Dev->>Pub: pastes live URL
    Pub-->>Bus: publish.succeeded.v1
```

Three things the earlier version of this diagram omitted, each of which changes what the system is:
the **Reviewer** and the **gate** both run before a human ever sees a draft; the **approval is only
valid if the maintainer edited something**; and the `draft`-tier channel is fully tracked despite
never being automated.

## 2. Wrong post → retraction

The path nobody diagrams and everybody eventually walks.

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Maintainer
    participant CLI as /eadros retract
    participant Store as SQLite store
    participant A as Dev.to (delete: edit-only)
    participant B as Hacker News (delete: unsupported)

    Dev->>CLI: retract --id p-42 --class leak
    CLI->>Store: /eadros pause — block every dispatch first
    Note over CLI,Store: a leak usually means the gate missed a class;<br/>the queue behind it may carry more
    CLI->>Store: read posts, tier_at_publish, channel api.delete
    CLI-->>Dev: plan per channel — including what cannot be undone
    Dev->>CLI: confirm
    CLI->>A: unpublish + correct in place
    A-->>CLI: ok
    CLI->>B: (no delete endpoint exists)
    CLI-->>Dev: removal impossible here — draft a correction comment<br/>and fix the artifact the post pointed at
    CLI->>Store: post.retracted.v1 + gate_verdict_id that PASSED this content
    Note over Store: the verdict becomes a test case in /eadros eval —<br/>the only way the gate improves from being wrong
    Dev->>CLI: /eadros resume (kill-switch owner only)
    CLI->>Store: re-gate the paused queue under the updated rules
```

The asymmetry is the point: on one channel the mistake is undone in seconds, on the other it is
permanent. Knowing which is which **before** publishing is what `api.delete` is for
([ADR-0011](../adr/ADR-0011-channel-capability-tiers.md)), and it is why the gate is strictest on
channels where retraction does not exist.
