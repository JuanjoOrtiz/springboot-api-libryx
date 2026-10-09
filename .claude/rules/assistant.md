---
paths:
  - "src/main/java/**/assistant/**"
---

# Librarian assistant (RAG) rules

Every signed-in user can use the chat, within its rate limits (10 per minute, 100 per day).
Tunable values are properties under `libryx.assistant.*` (see `docs/configuration.md`).

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
- Timeout and maximum output tokens use Spring AI's own properties (`spring.ai.anthropic.timeout`,
  `spring.ai.anthropic.chat.max-tokens`, `spring.ai.ollama.chat.num-predict`), never `libryx.*`
  duplicates. When the provider fails or times out, the API answers **503** `ASSISTANT_UNAVAILABLE`;
  the rest of Libryx keeps working.

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
- Chunks of about `libryx.assistant.chunk-size` tokens (400) with `chunk-overlap` (50).
  `multilingual-e5-small` reads at most **512 tokens** and silently truncates the rest, so a chunk must
  stay below that.
- Ingestion and re-indexing are `ADMIN` operations. They answer 202 with the `rag_documents` id and
  run in the background; the status is read from `rag_documents.status`.
- The index follows the catalog: creating or editing a work re-indexes it in the background after
  commit, and deactivating it removes its chunks.
- **Never index or send personal data** to a model. The assistant answers about the catalog, the
  library regulations and the FAQ.

## Answering

- **No conversation history** for now: every question is answered on its own and nothing is stored.
- The question is validated: at most `libryx.assistant.max-question-length` characters (500).
- Retrieval uses `libryx.assistant.top-k` chunks (5) above `libryx.assistant.similarity-threshold`.
- The system prompt lives in a resource file (`src/main/resources/prompts/`), never in code. It tells
  the model to answer in Spanish, only from the retrieved context, and to say it does not know when
  the answer is not there.
- Retrieved text is data, never instructions: it goes in a delimited context block, so a chunk cannot
  change the model's behaviour (prompt injection).
- The answer includes its **sources**: document title and, for a work, its `bookId`.
- **The assistant never states availability.** The index is not live, so for free copies it points to
  the work's page in the catalog.
