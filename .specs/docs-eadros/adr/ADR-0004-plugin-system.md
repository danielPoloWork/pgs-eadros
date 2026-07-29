# ADR-0004: Extensible Plugin System for Social & Dev Channels

* **Status:** Approved
* **Date:** 2026-07-28
* **Context:** New developer platforms (Dev.to, Hashnode, LinkedIn, Reddit, X/Twitter, Discord, Mastodon) emerge continuously. Hardcoding integrations limits community contributions.
* **Decision:** EADROS will implement a Plugin System (`EadrosPlugin`) defining standard interfaces for Data Observers, Content Formatter Adapters, and Publishing Adapters.
* **Consequences:**
  * Positive: Easy community extension for new channels or git hosts (GitLab, Bitbucket).
  * Negative: Requires strict plugin API versioning and isolation.
