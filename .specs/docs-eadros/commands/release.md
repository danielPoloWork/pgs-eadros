# `/eadros release` — release announcements

Fires on `dev.eadros.git.release_published.v1`, or runs manually against a tag. Drafts the
announcement for each configured channel.

Releases are the one event that is unambiguously newsworthy, needs no mining, and carries its own
source material. They are also the easiest place to produce the promotional register this project
exists to avoid — *"Check out our new release v1.4!"* is a release post, and it is the example the
problem statement opens with.

## Procedure

1. **Read the release.** Tag, notes, and the commit range since `previous_tag`.
2. **Classify what actually changed**, deterministically, from the range: breaking changes, new
   capabilities, fixes with linked issues, performance deltas with benchmark records.
3. **Rank by reader impact, not by changelog order.** A changelog is ordered by convention; a post is
   ordered by what a reader needs first. Breaking changes lead — someone deciding whether to upgrade
   needs the cost before the features.
4. **Draft**, archetype `release`, through the normal pipeline and gate.
5. **Queue for review.**

## What separates this from a changelog

The pipeline is required to answer **why**, and it has the material to do so:

- A breaking change links to the ADR or PR that justifies it, so the post explains the trade-off
  rather than announcing the inconvenience.
- A performance claim carries its benchmark run id, or it is not made
  ([claim discipline](../agents/copywriter.md)).
- A fix links its issue, so a reader can tell whether it was their bug.

A release post that restates the changelog has produced nothing the changelog did not already say —
and the changelog is already linked from the release page.

## Locale

Release announcements are the clearest case for `translated` mode: they are factual, their value is
information rather than voice, and a translation loses nothing. Opinion and post-mortem archetypes
are where `authored` earns its cost.

## Embargo

Releases are published by definition, so `safety.embargo`'s unmerged-branch and commit-age rules do
not apply. **The path allowlist and deny-terms still do** — release notes are written by humans in a
hurry and are a common place for an internal hostname or an unreleased feature name to appear.

A prerelease (`prerelease: true`) does not fire automatically. Announcing a release candidate to a
general audience is usually a mistake, and when it is not, it is a deliberate one:
`--include-prerelease`.

## Boundary

Drafts and queues. Never publishes. Refuses while paused.

## Flags

| Flag | Effect |
|---|---|
| `--tag <tag>` | Run manually against a specific release |
| `--include-prerelease` | Draft for a release candidate |
| `--dry-run` | Show the classified changes and the plan; generate nothing |
