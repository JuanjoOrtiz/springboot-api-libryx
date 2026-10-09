# ADR-0003: AI providers for the librarian assistant

- **Status:** Accepted
- **Date:** 2026-10-09

## Context

The assistant answers questions about the catalog, the regulations and the FAQ with RAG over
MariaDB's `VECTOR` type. Local development must work offline and for free; production needs a
reliable chat model; tests and CI must never call a real provider or need an API key. Vectors from
different embedding models are not comparable.

## Decision

- **Chat:** Claude through the Anthropic API in `prod`; Ollama on the developer's machine in `local`;
  a test double in `test`. Chosen with `spring.ai.model.chat`.
- **Embeddings:** `multilingual-e5-small` (384 dimensions) run **inside the JVM** with ONNX, the same
  model in every environment. Its files ship inside the Docker image.
- Code depends only on `ChatClient`, `EmbeddingModel` and `VectorStore`; provider classes stay in `config`.

## Alternatives considered

| Alternative | Why it was discarded |
|---|---|
| Ollama in production | Needs a GPU-capable host; not suitable for a small Fargate task |
| Hosted embeddings API (OpenAI, Voyage…) | Another paid key and a network call per chunk, and local vectors would differ from production |
| One provider everywhere (Claude locally too) | Costs money on every local question and needs a key on every developer machine |

## Consequences

- **Gains:** free offline development, identical vectors everywhere, provider change is configuration.
- **Costs:** the task needs memory for the model; local and production answers differ in quality.
- **Watch for:** a model change, which means a migration of the vector dimension and a full re-index.
