# ADR-0003: Specialized Multi-Agent Orchestration

* **Status:** Approved — **amended by [COMPONENTS.md](../architecture/COMPONENTS.md)**. The specialisation principle stands, but only **three** of the six roles below remained model-bearing (Angle, Copywriter, Reviewer). Story Finder, Publisher, Analytics and Planner moved into deterministic code, because each carries a *guarantee* rather than a judgment — see [ADR-0013](ADR-0013-cost-control-and-model-routing.md) and [ADR-0014](ADR-0014-deterministic-pre-publish-gate.md). The orchestrator is a **fixed DAG with models in the nodes**, not an agent that plans ([STATE_MACHINE.md](../architecture/STATE_MACHINE.md)).
* **Date:** 2026-07-28
* **Context:** A single monolithic prompt cannot handle story mining, technical framing, copywriting, platform-specific formatting, and quality review.
* **Decision:** We will implement a specialized multi-agent architecture with discrete roles: Story Finder Agent, Angle Agent, Copywriter Agent, Reviewer Agent, Publisher Agent, and Analytics Agent.
* **Consequences:**
  * Positive: Higher quality output, modular agent prompt maintenance, clear audit trail per agent step.
  * Negative: Increased LLM token usage and pipeline latency.
