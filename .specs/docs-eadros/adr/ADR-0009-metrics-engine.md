# ADR-0009: DevRel Analytics & Learning Engine

* **Status:** Approved — **amended by [ADR-0015](ADR-0015-attribution-methodology.md)**. The engine stands; the correlation-to-traffic claim below is replaced by directional lift, since the GitHub Traffic API's 14-day window and referrer stripping do not support per-post attribution.
* **Date:** 2026-07-28
* **Context:** Teams need to understand which technical angles generate traffic, clones, and contributions.
* **Decision:** We will implement an Analytics Engine (`DevRelAnalytics`) that correlates published posts with GitHub traffic API data (clones, unique visitors, referral paths, stargazers, forks).
* **Consequences:** Provides feedback to the Angle Agent to double down on high-performing topics.
