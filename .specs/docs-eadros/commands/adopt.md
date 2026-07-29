# `/eadros adopt` — brownfield intake for a repository that already publishes

The front door for a project with an **existing DevRel presence**: a live X or LinkedIn account, a
Dev.to backlog, past Show HN attempts, a newsletter, a dormant blog. Sibling of
[`/eadros init`](init.md), which frames a *new* program. Owned by the **devrel-architect** role.

`adopt` is **read-only**. It produces a presence map, a goal menu and an `adoption:` block in the
manifest — it migrates nothing, posts nothing, and deletes nothing.

## Why it is a separate command

An existing presence is not a blank program with some history attached. It carries three things
`init` has no way to handle:

- **A voice that already exists in public.** The maintainer's published posts are the best possible
  Phase 3 sample — better than anything they would write for the interview — and importing them is
  strictly superior to eliciting a voice from scratch.
- **Commitments already made.** A cadence readers expect, a series mid-run, a handle whose
  followers came for a specific thing. A program that ignores these reads as a takeover.
- **Evidence of what worked.** Past posts with real engagement are the only pre-existing signal the
  ranking model will ever get, and they are gone the moment the analytics window closes.

## Procedure

1. **Preflight.** `/eadros doctor`, plus read access to whichever channels the maintainer can point
   at. Where an API cannot enumerate history, ask for URLs — a partial map is useful, a wrong one is
   not.

2. **Presence map (read-only).** For each channel: whether it is live or dormant, the observed
   cadence, the post count, the top performers by whatever metric the platform exposes, and the
   credential status. Report **what you could not see**, explicitly — an unreadable channel is a gap
   in the map, not an absence of activity.

3. **Voice import — the highest-value step.** Offer to derive the voice fingerprint from the
   maintainer's **actual published posts** rather than from fresh samples, and prefer the ones that
   performed. Record provenance `inferred_from_sample` and show the fingerprint back in plain
   language, exactly as [`onboard`](onboard.md) does. Then still run the calibration loop: a
   fingerprint from historical posts is a strong hypothesis, not a finished profile — voices drift,
   and the maintainer may dislike how they used to write.

4. **The goal menu.** Adoption is scoped by an explicit, closed choice — never "adopt everything":

   | Goal | What it means |
   |---|---|
   | `govern` | Bring the existing presence under the manifest, gate and budget. Change nothing else |
   | `voice-import` | Derive and calibrate the voice profile from published work, and stop there |
   | `reposition` | Run the Wave 0 audit against the existing presence and act on the findings |
   | `resume` | The presence is dormant; restart a cadence deliberately |
   | `measure` | Instrument what exists — snapshots, UTM, back-fill — without changing output |

   Multiple goals are legal; an empty selection is not. Record them in `adoption.goals`.

5. **Write the `adoption:` block.** `goals`, `presence_map_ref` (where the read-only report was
   captured), and per-answer `provenance`. Adoption provenance lives **here**, not in
   `interview.provenance` — the two intakes settled different questions and must stay separately
   auditable.

6. **Route into the normal pipeline.** `adopt` never completes a program on its own. It hands off to
   [`/eadros onboard`](onboard.md) for the phases its goals leave unsettled — channels and tiers
   always, since **an existing account proves nothing about whether its automation was permitted.**
   A maintainer who has been auto-posting to a `draft`-tier channel needs to hear that now, stated
   plainly, along with what the exposure is.

7. **Confirm.** Present the presence map, the goals, and — separately — **anything the existing
   presence is doing that the tier model would not permit**. That list is the reason this command
   exists, and burying it inside a summary wastes it.

## Boundary

The agent **reads and proposes**; the human owns every change. `adopt` writes only the manifest's
`adoption:` block and the presence-map report. It does not delete posts, does not alter live
content, does not rotate credentials, and does not publish. Anything it finds that ought to be
retracted goes to [`/eadros retract`](retract.md) as a proposal the maintainer confirms.
