# ADR-0002: Event-Driven Architecture for Engineering Observability

* **Status:** Approved
* **Date:** 2026-07-28
* **Context:** Engineering activities occur asynchronously across GitHub webhooks, CLI executions, CI/CD pipeline results, and manual maintainer triggers.
* **Decision:** EADROS will use an Event-Driven Architecture (EDA) with an internal Event Bus (`eadros.events`). Activities like `commit.pushed`, `pr.merged`, `gate.intercepted`, `release.published` emit standard CloudEvents.
* **Consequences:**
  * Positive: Loose coupling between git observers, AI agents, and social network publishers.
  * Negative: Requires event schema management and idempotent handling.
