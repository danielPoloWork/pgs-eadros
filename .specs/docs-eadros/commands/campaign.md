# `/eadros campaign <id>` — run a milestone campaign

Runs a campaign defined in `campaigns[]` — a v1.0 launch, a Show HN, an Awesome-list submission, a
conference talk. Owned by the **devrel-architect** role.

Campaigns differ from the weekly cadence in one way that shapes everything: **they are the moments
you only get once.** A Show HN can be submitted once per project in any meaningful sense. That is why
every campaign carries a precondition, and why this command's most important behaviour is refusing to
run.

## Preconditions

```yaml
campaigns:
  - id: show-hn-v1
    trigger: manual
    channels: [hackernews]
    archetype: tradeoff
    precondition: "positioning.last_audit.score >= 80 and positioning.demo_ref != ''"
```

Evaluated against the **recorded** manifest state before anything else happens. An audit older than
30 days counts as absent — the README has moved on, and a stale pass is not a pass
([launch](launch.md)).

**A campaign with no precondition is how a project spends its one launch on a README that was not
ready.** The interview requires one for every campaign, and `/eadros audit` reports any that lack it.

When a precondition fails, the command **refuses and says which clause failed with its current
value**:

```
show-hn-v1 will not fire.
  positioning.last_audit.score >= 80   → 62   ✗
  positioning.demo_ref != ''           → ''   ✗
Run /eadros launch to see the 7 blocking findings.
```

Overriding requires `--force`, which is recorded in the campaign's ledger entry with the failed
clauses. The override exists — it is the maintainer's project — but it leaves a trace rather than
looking like a normal run.

## Procedure

1. **Evaluate the precondition.** Refuse, or continue.
2. **Check the tier**, and say what it means. For a `draft`-tier campaign — which Show HN always is —
   state plainly that EADROS will prepare the submission and **a human will post it**.
3. **Mine or select the source story.** A campaign may name a candidate, or mine within a window with
   the campaign's archetype forced.
4. **Draft**, through the normal pipeline and the normal gate. Campaigns get no shortcut around
   review; the moments that matter most are the worst ones to skip a check on.
5. **Queue for approval**, flagged as a campaign so it is not lost among cadence items.
6. **Hand off or dispatch**, per tier ([publish](publish.md)).
7. **Record** the campaign run, including any `--force` and its failed clauses.

## The Show HN case

The archetypal `draft`-tier campaign, and the one the tier model was designed around. EADROS
prepares:

- a title under 80 characters, no marketing verbs, no version number unless the version *is* the
  point;
- the clean URL — **no UTM**, because a tagged Show HN link reads as marketing and the audience
  penalises it ([hackernews profile](../orchestrator/channels/hackernews.yaml));
- the submission checklist: the demo works on a cold machine, the README's first screen answers what
  and for whom, the author is free for the next three hours to answer the thread.

Then it stops. The last item on that checklist is the one no scheduler can supply, and it is usually
the one that decides the outcome.

## Boundary

Refuses on an unmet precondition unless explicitly forced, and records the force. Drafts and queues;
never approves. Dispatches only where the tier permits and only after human approval. A `draft`-tier
campaign has no dispatch path at all.

## Flags

| Flag | Effect |
|---|---|
| `--force` | Run despite a failed precondition. Recorded with the failed clauses |
| `--dry-run` | Evaluate the precondition and show the plan; generate nothing |
| `--candidate <id>` | Use a specific mined candidate as the source |
