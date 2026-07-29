# `/eadros upgrade` — migrate the core and the manifest

Moves `.eadros-core/` and `devrel.yaml` to a newer version. Proposes a diff; the human applies it.

## Procedure

1. **Report the delta.** Current version, target version, and what changed between them — schema
   migrations, new interview questions, changed defaults, revised channel profiles, new prompt
   versions.

2. **Migrate the store.** Forward-only, each migration in a transaction, with `manifest_rev` bumped.
   **Back up the database file first** — the whole program state is one file
   ([ADR-0016](../adr/ADR-0016-local-first-single-file-store.md)), so the backup is a copy and the
   rollback is a restore.

3. **Migrate the manifest.** New keys arrive with defaults, marked provenance `defaulted` and
   **echoed for confirmation** — never applied silently. An assumed value that the maintainer never
   saw is the failure the provenance block exists to prevent, and an upgrade is where it is most
   tempting to skip.

4. **Re-verify channel policies.** Any profile whose `policy.verified_on` the new version updated is
   listed with what changed. **A tier that moved down — a channel that permitted automation and no
   longer does — is a blocking finding**, not a note. Continuing to auto-post under withdrawn
   permission is the failure [ADR-0011](../adr/ADR-0011-channel-capability-tiers.md) exists to
   prevent, and an upgrade is exactly when it would slip through.

5. **Flag prompt changes.** A new `prompt_version` invalidates the recorded eval baselines. The
   upgrade **does not silently adopt it**: it reports which agents changed and recommends
   `/eadros eval --since <old-version>` before the next drafting run.

6. **Confirm, then apply.** Present store migrations, manifest diff, policy changes and prompt
   changes as one reviewable set. Nothing is written until the maintainer accepts.

## New interview questions

A version that adds a question does **not** re-run the interview. It asks only the new question,
records it `asked`, and leaves every existing answer untouched. Re-interviewing a maintainer who
already answered is how a tool teaches people to dread its updates.

## Boundary

Backs up before touching the store. Proposes every change and applies none without confirmation.
Never rotates credentials, never publishes, never re-runs a completed interview phase.

## Flags

| Flag | Effect |
|---|---|
| `--check` | Report the delta and exit; change nothing |
| `--to <version>` | Target a specific version rather than the latest |
| `--backup <path>` | Where the pre-migration copy goes |
