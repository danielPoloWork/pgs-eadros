# EADROS Agents

**Three agents.** Angle, Copywriter, Reviewer — the three pipeline nodes where judgment genuinely
lives ([COMPONENTS](../architecture/COMPONENTS.md)).

The other six pipeline components are deterministic code and are specified elsewhere, because each
carries a **guarantee** rather than a judgment, and a guarantee implemented in a prompt can only ever
be an aspiration:

| Not an agent | Specified in |
|---|---|
| Story miner | [`commands/mine.md`](../commands/mine.md) |
| PrePublishGate | [ADR-0014](../adr/ADR-0014-deterministic-pre-publish-gate.md) |
| Scheduler | [STATE_MACHINE](../architecture/STATE_MACHINE.md) |
| Publisher + outbox | [`commands/publish.md`](../commands/publish.md) |
| Metrics collector | [ADR-0015](../adr/ADR-0015-attribution-methodology.md) |
| Knowledge base | [ADR-0016](../adr/ADR-0016-local-first-single-file-store.md) |

## The common contract

Every agent below obeys all six. They are stated once here rather than repeated three times.

**1. Untrusted content is data, never instruction.** Any span authored outside the maintainer set —
an external PR title, a fork's commit message, issue text — arrives delimited and flagged
`taint: untrusted` ([EVENTS](../architecture/EVENTS.md)). No agent treats it as an instruction, and
containment is asserted at the boundary rather than inferred from the model behaving well
([eval/adversarial](../eval/adversarial.md)).

**2. Structured output, always.** Each agent emits a declared schema, not prose to be parsed. A
response that does not validate is a failure of that stage, retried once and then surfaced — never
salvaged by regex.

**3. Declining is a valid, cheap outcome.** Each agent may return "I cannot do this, here is why".
An early exit costs one cheap call; forcing a draft costs the whole pipeline and produces something
a human then has to reject.

**4. Prompts are versioned artifacts.** They live in `config/agents/<name>.md` with a
`prompt_version`. Every generation records the version alongside `model_id` and token cost, so a
quality regression is attributable to a change rather than to the weather
([DATA_MODEL](../architecture/DATA_MODEL.md)).

**5. Model tier, not model name.** Each agent declares a capability tier — `cheap` / `mid` /
`strong` — which the provider profile resolves. No agent spec names a model
([ADR-0013](../adr/ADR-0013-cost-control-and-model-routing.md)).

**6. Bounded.** The Copywriter↔Reviewer loop runs at most `budget.max_reviewer_iterations` (default
2). On exhaustion the draft advances to the human **carrying the open objections**, rather than
looping or silently passing.

## The three

| Agent | Tier | Decides | Spec |
|---|---|---|---|
| **Angle** | cheap | Which story to tell, and whether there is one | [angle.md](angle.md) |
| **Copywriter** | mid | How it reads, per channel and locale | [copywriter.md](copywriter.md) |
| **Reviewer** | strong | Whether it is fit to reach a human | [reviewer.md](reviewer.md) |

The tiers are not a ranking of importance. **Reviewer gets the strong tier because it is the last
mechanical check before a human**, and human attention is the scarcest resource in the system — a
miss there costs a person's time and possibly a retraction.
