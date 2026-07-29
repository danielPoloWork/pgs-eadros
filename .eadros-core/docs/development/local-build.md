# Local Build & Test

How to build, test, and check `pgs-eadros` on your machine. CI runs the same commands
on Linux / Windows / macOS on Node 22, 24 LTS; reproducing them locally avoids a red round-trip.

## Prerequisites

- **TypeScript** toolchain.
- **Build system:** tsup (esbuild) / tsc --build.
- **Package manager:** pnpm (workspaces, locked).
- **Formatter / linter:** Prettier, ESLint (typescript-eslint, type-aware) + tsc --noEmit --strict.
- **Docs:** TypeDoc (for the API docs build).

## Commands

```bash
# Build
pnpm build

# Test
pnpm test

# Format check
prettier --check .

# Lint
eslint . --max-warnings 0 && tsc --noEmit

# Benchmark
pnpm vitest bench

# Cross-artifact congruence (run before drafting any PR)
python .eadros-core/tools/consistency_lint.py
```

## Before you open a PR

1. `prettier --check .` and `eslint . --max-warnings 0 && tsc --noEmit` are clean.
2. `pnpm test` passes; new/changed behavior is covered (≥ 80% line).
3. tsc --strict (type soundness), vitest --detectOpenHandles (leak/handle), eslint --max-warnings 0 are green where applicable.
4. `python .eadros-core/tools/consistency_lint.py` passes.
5. The relevant docs (README, ROADMAP, ADRs, patterns, changelog) are updated in the same
   PR — see [`../workflow/documentation.md`](../workflow/documentation.md).
