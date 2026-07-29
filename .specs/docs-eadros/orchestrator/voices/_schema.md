# Voice profile schema

One file per writing voice: `voices/<slug>.yaml`. A voice profile is the machine-readable record
of **how a specific maintainer writes**, extracted from their own writing rather than described in
adjectives. It is the sibling of an EADOS domain profile — it constrains everything downstream —
and it carries the same stance:

> **There is no unsupported voice, only a voice not yet profiled.**

## Why samples, not adjectives

A maintainer who says *"professional but approachable"* has specified nothing. Every model already
believes it writes that way, and the result is the uniform register that developers recognise and
reject on sight. That recognition is the whole risk of this product: the thesis is authenticity,
and a detectable generated voice falsifies it in one post.

Adjectives are unfalsifiable; a fingerprint is measurable. **Mean sentence length is checkable. A
forbidden construction is checkable. "Approachable" is not.** So Phase 3 of the interview asks for
two or three things the maintainer has written and derives the profile from them, recording every
derived field with provenance `inferred_from_sample` — and showing it back in plain language
before adopting it, because an inferred value the maintainer never saw is indistinguishable from
a guess.

## Fields

```yaml
voice:          engineer-practitioner
display_name:   ""
person:         person              # person ("I built…") | project ("v2.0 ships…")
byline:         ""
stance:         ""                  # one line: the posture the writing takes toward the reader

fingerprint:                        # extracted at Q3.1; every field is mechanically checkable
  mean_sentence_words:  { target: 18, tolerance: 6 }
  sentence_variance:    high        # low | medium | high — uniform sentence length is the single
                                    # loudest generated-prose tell
  paragraph_max_sentences: 4
  jargon_density:       high        # low | medium | high — assumed reader knowledge
  hedging:              low         # low | medium | high — "arguably", "it could be said"
  code_to_prose_ratio:  { min: 0.15 }
  formatting:
    bullets:    sparing             # never | sparing | frequent
    headers:    true
    em_dash:    true
    emoji:      none                # none | sparing | frequent
  opening_move: [claim, problem, number]   # the openers this voice actually uses

forbidden:
  words:         []                 # the obvious half
  constructions: []                 # the half that matters more — see below

archetypes:                         # Q3.3 consent. postmortem and opinion default OFF: publishing
  postmortem:  { enabled: false, max_per_month: 0 }   # a failure story or a stated opinion under
  opinion:     { enabled: false, max_per_month: 0 }   # someone's name is a reputational act that
  subtraction: { enabled: true,  max_per_month: 2 }   # must be consented to, never assumed
  tradeoff:    { enabled: true,  max_per_month: 2 }
  benchmark:   { enabled: true,  max_per_month: 2, requires_evidence: true }
  buildlog:    { enabled: true,  max_per_month: 4 }
  howto:       { enabled: true,  max_per_month: 2 }
  release:     { enabled: true,  max_per_month: 4 }

claim_discipline: strict            # strict | relaxed. strict = every factual claim carries a
                                    # resolvable source_ref (SHA, PR, file:line, benchmark run id)
                                    # and code snippets are extracted verbatim at a SHA. The
                                    # Reviewer resolves each ref mechanically; an unresolvable
                                    # claim FAILS the draft rather than being softened.

disclosure:     { mode: standing_line, text: "", per_channel: true }

calibration:    []                  # [{date, source_ref, rounds, accepted}]
```

## The forbidden list is a lint, not a judgment

`forbidden.words` is the easy half and the one everyone writes. `forbidden.constructions` is where
generated prose actually gives itself away, and it is what makes the Reviewer Agent's "zero
buzzwords" rule enforceable without asking a model to judge taste:

- the **rhetorical-question opener** (*"Ever wondered why…?"*)
- the **antithesis** (*"It's not X — it's Y"*)
- the **scene-setting preamble** (*"In today's fast-paced world…"*)
- the **parallel triad** — three balanced items where the content only supports two
- **emoji section headers**
- the **summary-of-the-summary** closing paragraph that restates the post
- **hedged authority** — a confident claim wrapped in "arguably" so it cannot be wrong

These run as a deterministic check before a human ever sees the draft. A hit is a rewrite, not a
warning: the whole point of putting them in a file is that the rule survives a model change.

## The calibration loop

A profile written from samples is a hypothesis. The loop that makes it a measurement runs at the
end of Phase 3 and is re-runnable with `/eadros onboard --recalibrate`:

1. Take **one real recent commit** from the repository.
2. Generate **three short drafts** under the current profile.
3. The maintainer picks one and **edits it freely**.
4. **Diff the edit against the draft** and fold the delta into the fingerprint.

**What they cut is worth more than what they kept.** Deletions reveal the constructions the voice
rejects — which is exactly what `forbidden.constructions` needs and what no amount of asking
produces. Two rounds is normal; converging on the first round is suspicious and usually means the
maintainer was being polite rather than satisfied.

Each round appends to `calibration[]`, so the profile carries its own evidence.

## Shipped profiles

| Profile | Person | Stance |
|---|---|---|
| [`engineer-practitioner.yaml`](engineer-practitioner.yaml) | `person` | Writes from having built the thing; assumes a peer reader |

A seed, not the allowed set. `architect-essayist`, `terse-maintainer` and `builder-in-public` are
the next profiles to author — from real samples, like this one.
