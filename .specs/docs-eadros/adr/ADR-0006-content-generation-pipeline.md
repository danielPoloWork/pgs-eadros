# ADR-0006: Multi-Stage Content Generation Pipeline

* **Status:** Approved
* **Date:** 2026-07-28
* **Context:** Raw git diffs and commit messages cannot be pasted directly to social channels. They require extraction, framing, draft generation, and platform formatting.
* **Decision:** We define a 4-stage pipeline: 
  `Mining -> Angle Selection -> Channel Drafting -> Validation & Polish`.
* **Consequences:** Ensures each piece of content has a distinct technical hook before drafting.
