# EADROS Architecture: System Context (C4 Level 1)

```mermaid
graph TD
    User["Open Source Maintainer / DevRel Lead"]
    GitHub["GitHub / GitLab Repository"]
    EADROS["EADROS (DevRel OS Engine)"]
    LLMProvider["LLM Providers (Anthropic / OpenAI / Ollama)"]
    SocialChannels["Dev Platforms (LinkedIn, Dev.to, Hashnode, Reddit, X, Discord)"]

    User -->|Configures & Approves Content| EADROS
    GitHub -->|Webhooks & Git Events| EADROS
    EADROS -->|Prompts & Context| LLMProvider
    LLMProvider -->|Generated Narratives| EADROS
    EADROS -->|Publishes Approved Content| SocialChannels
    SocialChannels -->|Engagement Data| EADROS
    EADROS -->|Analytics & Traffic Reports| User
```

## System Boundaries
* **Input Boundary:** Ingests Git logs, Webhooks, GitHub REST/GraphQL API, and Markdown documentation.
* **Processing Boundary:** Event Bus, Agent Orchestrator, LLM Provider Abstraction, Vector KB.
* **Output Boundary:** Human Review CLI/Dashboard, Publisher Adapters (LinkedIn, Dev.to, Hashnode, Reddit, X, Discord).
