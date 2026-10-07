---
paths:
  - "src/main/java/**/assistant/**"
---

# Librarian assistant (RAG) rules

## Providers

Chosen per profile with `spring.ai.model.chat` and `spring.ai.model.embedding`, always set explicitly.

| Function | Profile | Provider | Starter | Selection |
|---|---|---|---|---|
| Chat | `local` | Ollama | `spring-ai-starter-model-ollama` | `spring.ai.model.chat=ollama` |
| Chat | `prod` | Claude (Anthropic) | `spring-ai-starter-model-anthropic` | `spring.ai.model.chat=anthropic` |
| Chat | `test` | None: test double | — | `spring.ai.model.chat=none` |
| Embeddings | `local` and `prod` | ONNX model inside the JVM | `spring-ai-starter-model-transformers` | `spring.ai.model.embedding=transformers` |
| Embeddings | `test` | None: test double | — | `spring.ai.model.embedding=none` |

- Code depends on `ChatClient`, `EmbeddingModel` and `VectorStore`. **Never** import a provider-specific
  class outside `config`: switching provider is a configuration change.
- The chat model name goes in `spring.ai.ollama.chat.model` and `spring.ai.anthropic.chat.model`,
  never in code. The Anthropic key (`spring.ai.anthropic.api-key`) exists only in `prod` and is a secret.
- **Tests and CI never call a provider or load the model**: they use doubles of `ChatModel` and
  `EmbeddingModel`. There are no AI keys in CI.
- Timeout and maximum output tokens are properties. When the provider fails or times out, the API
  answers **503** `ASSISTANT_UNAVAILABLE`; the rest of Libryx keeps working.

## Embeddings

- Multilingual model `multilingual-e5-small` (384 dimensions) in ONNX format, the **same in every
  environment**: vectors from different models are not comparable.
- E5 models need a prefix: `passage: ` when indexing a chunk and `query: ` when searching. Apply it in
  a single place in this module.
- The model files are not versioned in git: they are fetched by the script documented in the README
  and baked into the Docker image. In production they are **not** downloaded at startup.
- Changing the model means a migration for the vector dimension and re-indexing everything.

## Vector store and ingestion

- `MariaDBVectorStore` on the `vector_store` table, `VECTOR(384)`, created by **Flyway**
  (`spring.ai.vectorstore.mariadb.initialize-schema=false`, `dimensions=384`).
- `rag_documents` records each source (`BOOK`, `REGULATION`, `FAQ`, `MANUAL`) with its `content_hash`
  (SHA-256) and `embedding_model`, so only what changed is re-indexed.
- Chunk metadata: `document_id`, `book_id`, `chunk_index`, `source_type`.
- Ingestion and re-indexing are `ADMIN` operations, answer 202 and run in the background.
- **Never index or send personal data** to a model. The assistant answers about the catalog, the
  library regulations and the FAQ.
- Chunking strategy, conversation history and source citations are **not defined yet: ask before
  implementing them**.
