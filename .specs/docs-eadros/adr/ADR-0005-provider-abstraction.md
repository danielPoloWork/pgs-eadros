# ADR-0005: Multi-LLM Provider Abstraction Layer

* **Status:** Approved
* **Date:** 2026-07-28
* **Context:** Open source users run different AI models (Claude 3.5, OpenAI GPT-4o, Local Ollama/Llama 3, DeepSeek). EADROS must not be vendor-locked.
* **Decision:** We will build a unified Provider Abstraction Layer (`LLMProvider`) supporting Anthropic, OpenAI, Ollama, and LiteLLM out-of-the-box.
* **Consequences:**
  * Positive: Users can run EADROS completely locally (Ollama) or with top-tier API providers.
  * Negative: Requires normalizing prompt schemas and handling model-specific token limits.
