# ADR-0008: Human-in-the-Loop Approval & Governance Gate

* **Status:** Approved — **amended by [ADR-0014](ADR-0014-deterministic-pre-publish-gate.md)**. The gate stands; the "Consequences" claim below that it *eliminates* hallucination risk is withdrawn, as human attention is not a control.
* **Date:** 2026-07-28
* **Context:** Fully automated posting risks publishing incorrect technical claims or confidential repo details.
* **Decision:** EADROS will enforce a Human Review Gate (`HumanReviewGate`). Generated content is queued in a local Markdown dashboard, CLI prompt, or Discord/Telegram approval channel. Nothing is published without explicit human confirmation.
* **Consequences:** Eliminates hallucination risk and protects brand reputation.
