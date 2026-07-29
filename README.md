# pgs-eadros

> Governed agent pipeline that turns real engineering activity into human-approved developer-relations posts.

![Status](https://img.shields.io/badge/Status-v0.0.0-blue)

Part of the **Production-Grade Systems** series. A
cli written in **TypeScript**, built and governed to an enterprise quality
bar: full CI matrix, static analysis, sanitizers, documented design decisions, and SemVer
releases.

## What it is

Transform every engineering activity into discoverable, high-impact technical content
(vision/VISION.md). EADROS observes real engineering activity in a repository — commits,
pull requests, releases, ADR changes, CI gate intercepts — identifies the events that carry
a technical story, and runs a governed agent pipeline that drafts, checks and distributes
it. A human approves everything that ships.

It exists because high-quality open-source projects fail from absent distribution, not
absent quality: maintainers have no time for Developer Relations, and generic social-media
tooling produces the promotional register developer communities reject on sight.

Three constraints shape the system, and none of them is "generate good text" — text
generation is the commodity part. (1) Most destinations forbid what is being automated, and
over-automation can get the promoted project's domain permanently banned from the
highest-signal channel in its ecosystem (ADR-0011). (2) Detectably generated output
falsifies the entire thesis in one post, unrecoverably — the product is authenticity, so a
recognisable model register is a category failure, not a quality issue (ADR-0012). (3) The
pipeline reads private repositories, holds publishing credentials, and ingests text any
stranger can write on a public repository's pull request (ADR-0014).

Scope: one repository per program, a solo maintainer or small team, at most two substantive
posts per week, 6-8 channels. Local-first: one CLI, one SQLite file, no servers.

The frozen specification is in
[`docs/specs/01_spec_eadros.md`](docs/specs/01_spec_eadros.md).

## Build, test, run

```bash
pnpm build
pnpm test
```

- **Toolchain:** tsup (esbuild) / tsc --build, Vitest (or Jest), Prettier, ESLint (typescript-eslint, type-aware) + tsc --noEmit --strict.
- **Supported platforms:** Linux / Windows / macOS on Node 22, 24 LTS.
- Consumers import the public surface via: `import { ... } from '@d4np/eadros';`.

See [`docs/development/local-build.md`](docs/development/local-build.md) for the full local
setup.

## How this project is run

| Document | Purpose |
|---|---|
| [`AGENTS.md`](AGENTS.md) | How AI agents (and humans) work in this repo — the contract. |
| [`ROADMAP.md`](ROADMAP.md) | The numbered plan and what is done. |
| [`docs/adr/`](docs/adr/) | Why it is built the way it is (Architecture Decision Records). |
| [`docs/patterns/`](docs/patterns/) | Design patterns adopted, rejected, or considered. |
| [`docs/workflow/`](docs/workflow/) | Git, documentation, release, and maintenance conventions. |
| [`CHANGELOG.md`](CHANGELOG.md) | User-visible changes per release. |
| [`SECURITY.md`](SECURITY.md) | How to report a vulnerability. |

## Milestones

| # | Title | Status |
|---|---|---|
| 1 | Project bootstrap & CI | ⏳ in progress |
| 2 | Foundation | ⏳ planned |
| 3 | Wave 0 positioning | ⏳ planned |
| 4 | Mining and drafting | ⏳ planned |
| 5 | Gate and publish, auto tier | ⏳ planned |
| 6 | assisted and draft tiers | ⏳ planned |
| 7 | Measurement | ⏳ planned |
| 8 | Verification harness | ⏳ planned |
| 9 | Release and distribution | ⏳ planned |


## License

MIT © 2026 Daniel Polo. See [`LICENSE`](LICENSE).
