# `/eadros eval` — run the verification suites

Runs the golden-set and property suites in [`eval/`](../eval/README.md). Owned by the
**reviewer** role. Read-only: it generates no content, contacts no platform, and publishes nothing.

## Procedure

1. **Resolve the suite set.** `--suite <name>` or all. Suites are independent and may run in any
   order; none depends on another's state.
2. **Resolve the provider.** Deterministic suites need none. Generative suites use **recorded
   fixtures by default** and live providers under `--live` — CI uses fixtures; the nightly job uses
   `--live`, because a suite that only ever sees fixtures eventually measures the fixtures.
3. **Run.** Deterministic suites once; generative suites `--runs N` times per case (default 20).
4. **Score per class.** Never aggregate. Report mean, standard deviation and the lower confidence
   bound for generative suites; exact values for deterministic ones.
5. **Compare.** Against the recorded baseline for the current `prompt_version`, and against each
   suite's declared trivial baseline.
6. **Gate.** Deterministic suites fail on any regression. Generative suites fail when a **per-class
   lower bound** crosses its threshold. The `adversarial` suite's containment and secrets classes
   fail on anything below 1.0, with no threshold negotiation.
7. **Record.** Write results against `prompt_version`, corpus version and the model ids used, so a
   regression is attributable to a change rather than to drift.

## Reading the report

```
suite        class                  metric  mean    σ      lcb    gate   verdict
reviewer     hype_word              recall  0.99   0.01   0.98   0.98   pass
reviewer     construction           recall  0.91   0.04   0.87   0.85   pass
reviewer     false_technical        recall  0.58   0.09   0.51   0.60   FAIL
reviewer     clean                  fp      0.06   0.02   ─      0.10   pass
                                                          vs baseline (regex): +0.31 overall
adversarial  injection_containment  rate    1.00   0.00   1.00   1.00   pass
miner        strong_recall          recall  0.93    ─      ─     0.90   pass
cost         campaign_cost           €      0.079   ─      ─     0.072  FAIL  (+21% vs v3)
```

Two rows fail; the run fails. **There is no aggregate score to average them away** — that is the
point of the layout. `false_technical` regressing while every other reviewer class passes is the
exact shape of a change that looks like an improvement, and a summary number would hide it.

The `vs baseline` line answers the question a per-class table does not: is this component still
worth its cost? A reviewer at +0.31 over a regex is earning its keep; one at +0.03 is not, whatever
its absolute numbers say ([`eval/README.md`](../eval/README.md)).

## High-variance cases

Cases whose verdict flips across runs are listed separately, never averaged into a class score:

```
unstable (verdict flipped across 20 runs):
  rev-0142  false_technical   12/20 detected
  rev-0177  construction      9/20 detected
```

These are where prompt changes produce phantom improvements. A "gain" that comes entirely from
unstable cases resolving favourably in one run is noise, and surfacing them by name is what stops it
being reported as progress.

## Corpus maintenance

`--add-from-retractions` ingests every `retract` since the last run into the `adversarial` corpus,
carrying `gate_verdict_id` and asserting the content is now blocked. `--add-from-rejections` does
the same for human rejections into the `reviewer` corpus as labelled false negatives.

Both are the production feedback loop closing: the system's own failures become its regression
tests, which is the only way the gate improves from having been wrong.

## Boundary

Read-only. `eval` writes only to the results store and, with the `--add-*` flags, to the corpus.
It never modifies prompts, weights or the manifest — a suite that tunes the thing it measures is not
a measurement.

## Flags

| Flag | Effect |
|---|---|
| `--suite <name>` | One suite; default all |
| `--runs <n>` | Runs per case for generative suites (default 20) |
| `--live` | Live providers instead of recorded fixtures |
| `--baseline` | Also score each suite's trivial baseline and print the delta |
| `--since <prompt_version>` | Compare against a recorded run |
| `--add-from-retractions` / `--add-from-rejections` | Grow the corpus from production failures |
| `--ci` | Non-zero exit on any gate failure; machine-readable output |
