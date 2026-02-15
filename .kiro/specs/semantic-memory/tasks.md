# Implementation Plan: Project Cortex (Semantic Memory & Learning)

## Overview

Implement the Cortex semantic memory system incrementally: data models first, then storage, embedding, search, indexing, recall, API, and finally frontend integration. Each step builds on the previous and includes testing sub-tasks.

## Tasks

- [ ] 1. Create data models and configuration
  - [ ] 1.1 Create `openhands/memory/cortex/` package with `__init__.py`
    - Create directory structure: `openhands/memory/cortex/`
    - _Requirements: N/A (setup)_

  - [ ] 1.2 Implement `MemoryEntry` and `KnowledgeCategory` data models
    - Create `openhands/memory/cortex/models.py`
    - Define `KnowledgeCategory` enum with values: `style_preference`, `architectural_decision`, `bug_fix`, `code_pattern`, `project_context`, `workflow`
    - Define `MemoryEntry` Pydantic model with fields: `id`, `content`, `category`, `metadata`, `embedding`, `created_at`, `updated_at`, `source_conversation_id`
    - UUID default for `id`, `datetime.utcnow` defaults for timestamps
    - _Requirements: 1.1, 6.1, 6.2_

  - [ ] 1.3 Write property test for MemoryEntry serialization round trip
    - **Property 14: Serialization round trip**
    - Use Hypothesis to generate random valid MemoryEntry instances
    - Verify `MemoryEntry.model_validate_json(entry.model_dump_json()) == entry`
    - **Validates: Requirements 6.3**

  - [ ] 1.4 Implement `CortexConfig` Pydantic model
    - Create `openhands/core/config/cortex_config.py`
    - Fields: `enabled`, `embedding_model`, `embedding_dimensions`, `embedding_max_tokens`, `default_relevance_threshold`, `max_memory_context_tokens`, `deduplication_threshold`, `storage_path`
    - Add field validators for `embedding_dimensions` (positive), `default_relevance_threshold` (0.0–1.0)
    - Add `cortex: CortexConfig` field to `OpenHandsConfig`
    - _Requirements: 7.1, 7.2, 7.3_

  - [ ] 1.5 Write property test for CortexConfig validation
    - **Property 15: Invalid config raises validation error**
    - Use Hypothesis to generate invalid config values (negative dimensions, threshold > 1.0, etc.)
    - Verify `ValidationError` is raised
    - **Validates: Requirements 7.3**

- [ ] 2. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 3. Implement EmbeddingProvider
  - [ ] 3.1 Implement `EmbeddingProvider` abstract base class and `LiteLLMEmbeddingProvider`
    - Create `openhands/memory/cortex/embedding.py`
    - Define abstract `EmbeddingProvider` with `embed(text) -> list[float]` and `embed_batch(texts) -> list[list[float]]`
    - Implement `LiteLLMEmbeddingProvider` using `litellm.embedding()`
    - Handle text truncation when exceeding `max_tokens`
    - Wrap litellm errors with descriptive messages
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5_

  - [ ] 3.2 Write property test for embedding dimensionality
    - **Property 5: Embedding dimensionality invariant**
    - Mock litellm to return vectors of configured dimensions
    - Use Hypothesis to generate random text strings
    - Verify output vector length equals configured dimensions
    - **Validates: Requirements 2.1**

  - [ ] 3.3 Write property test for embedding error propagation
    - **Property 7: Embedding error propagation**
    - Mock litellm to raise various errors
    - Verify the raised exception contains a descriptive message
    - **Validates: Requirements 2.4**

- [ ] 4. Implement MemoryStore
  - [ ] 4.1 Implement `MemoryStore` abstract base class
    - Create `openhands/memory/cortex/store.py`
    - Define abstract methods: `save`, `get`, `delete`, `list_entries`, `search`, `exists`
    - _Requirements: 1.1, 1.2, 1.3, 1.5, 3.1, 3.2, 3.3, 3.4_

  - [ ] 4.2 Implement `FileMemoryStore`
    - Create `openhands/memory/cortex/file_memory_store.py`
    - Store entries as JSON files via `FileStore` backend at `{storage_path}/{user_id}/{entry_id}.json`
    - Maintain in-memory numpy index for cosine similarity search
    - Implement index rebuild on init, incremental update on add/delete
    - Implement duplicate ID check on save (raise `DuplicateEntryError`)
    - Implement cosine similarity search with top_k, threshold, and category filtering
    - _Requirements: 1.1, 1.2, 1.3, 1.5, 3.1, 3.2, 3.3, 3.4, 3.5_

  - [ ] 4.3 Write property test for store-retrieve round trip
    - **Property 1: Store-retrieve round trip**
    - Use Hypothesis to generate random MemoryEntry instances with embeddings
    - Store, then retrieve by ID, verify equivalence
    - **Validates: Requirements 1.2**

  - [ ] 4.4 Write property test for duplicate ID rejection
    - **Property 2: Duplicate ID rejection**
    - Generate a random entry, store it, then attempt to store another with the same ID
    - Verify error is raised and original entry is unchanged
    - **Validates: Requirements 1.3**

  - [ ] 4.5 Write property test for delete removes entry
    - **Property 4: Delete removes entry**
    - Store a random entry, delete it, verify retrieval raises not-found error
    - **Validates: Requirements 1.5**

  - [ ] 4.6 Write property test for search ordering
    - **Property 8: Search results ordered by descending score**
    - Store multiple entries with random embeddings, search, verify results are in non-increasing score order
    - **Validates: Requirements 3.1**

  - [ ] 4.7 Write property test for search top_k and threshold
    - **Property 9: Search respects top_k and threshold**
    - Store entries, search with random top_k and threshold values
    - Verify result count <= top_k and all scores >= threshold
    - **Validates: Requirements 3.2, 3.3**

  - [ ] 4.8 Write property test for search category filter
    - **Property 10: Search category filter**
    - Store entries with mixed categories, search with a category filter
    - Verify all results match the filter category
    - **Validates: Requirements 3.4**

- [ ] 5. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 6. Implement CortexService
  - [ ] 6.1 Implement `CortexService` orchestrator
    - Create `openhands/memory/cortex/service.py`
    - Implement `add_memory`: generate embedding via provider, create MemoryEntry, persist via store. If provider fails, store with null embedding.
    - Implement `search_memories`: embed query, delegate to store search
    - Implement `delete_memory`, `get_memory`, `list_memories` as pass-through to store
    - _Requirements: 1.1, 1.4, 3.1, 3.2, 3.3, 3.4, 8.1, 8.2, 8.3, 8.4_

  - [ ] 6.2 Write property test for storage without embedding when provider unavailable
    - **Property 3: Storage without embedding when provider unavailable**
    - Mock EmbeddingProvider to raise errors
    - Verify entry is stored with null embedding and is retrievable
    - **Validates: Requirements 1.4**

  - [ ] 6.3 Write unit tests for CortexService
    - Test add_memory happy path
    - Test search_memories with various parameters
    - Test delete and get operations
    - _Requirements: 1.1, 1.4, 3.1_

- [ ] 7. Implement CortexIndexer
  - [ ] 7.1 Implement `CortexIndexer`
    - Create `openhands/memory/cortex/indexer.py`
    - Implement `index_conversation`: send conversation events to LLM with extraction prompt, parse structured output into MemoryEntry records
    - Implement deduplication: before storing, search for similar entries above `deduplication_threshold`, update timestamp if found
    - Handle unparseable content: log warning, skip segment, continue
    - _Requirements: 4.1, 4.2, 4.3, 4.4_

  - [ ] 7.2 Write property test for valid category assignment
    - **Property 11: Indexed entries have valid categories**
    - Mock LLM to return entries with various categories
    - Verify all categories are valid KnowledgeCategory enum values
    - **Validates: Requirements 4.2**

  - [ ] 7.3 Write property test for deduplication
    - **Property 12: Deduplication on re-index**
    - Store an entry, then index semantically identical content
    - Verify store size does not increase
    - **Validates: Requirements 4.3**

- [ ] 8. Implement CortexRecall
  - [ ] 8.1 Implement `CortexRecall`
    - Create `openhands/memory/cortex/recall.py`
    - Implement `get_memory_context`: query CortexService, format results as structured text block, respect token budget by removing lowest-scoring entries
    - _Requirements: 5.1, 5.2, 5.3, 5.4_

  - [ ] 8.2 Integrate CortexRecall into the `Memory` class
    - Modify `openhands/memory/memory.py` to call `CortexRecall.get_memory_context()` during workspace context recall (in `_on_workspace_context_recall`)
    - Only activate when `cortex.enabled` is True in config
    - _Requirements: 5.1, 5.2_

  - [ ] 8.3 Write property test for memory context formatting and token budget
    - **Property 13: Memory context formatting and token budget**
    - Generate random sets of (MemoryEntry, score) pairs and random token budgets
    - Verify formatted output does not exceed budget and contains highest-scoring entries in order
    - **Validates: Requirements 5.2, 5.3**

- [ ] 9. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 10. Implement Cortex API endpoints
  - [ ] 10.1 Create Cortex API router
    - Create `openhands/server/routes/cortex.py`
    - Implement endpoints:
      - `GET /api/cortex/memories` — list memories (paginated, optional category filter)
      - `GET /api/cortex/memories/{id}` — get single memory
      - `DELETE /api/cortex/memories/{id}` — delete memory
      - `POST /api/cortex/memories/search` — search by query text
    - Follow existing router patterns in `openhands/server/routes/`
    - Mount router in the server app
    - _Requirements: 8.1, 8.2, 8.3, 8.4_

  - [ ] 10.2 Write property test for API pagination
    - **Property 16: API pagination invariant**
    - Generate random sets of entries and page_size values
    - Verify returned page has at most page_size entries
    - **Validates: Requirements 8.1**

  - [ ] 10.3 Write unit tests for API endpoints
    - Test list, get, delete, search endpoints
    - Test error responses (404 for missing entries, 400 for invalid params)
    - _Requirements: 8.1, 8.2, 8.3, 8.4_

- [ ] 11. Integrate indexing into session lifecycle
  - [ ] 11.1 Wire CortexIndexer into session completion
    - Hook into the session/conversation completion flow to trigger `CortexIndexer.index_conversation()`
    - Only activate when `cortex.enabled` is True
    - _Requirements: 4.1_

  - [ ] 11.2 Write unit tests for session integration
    - Test that indexer is called on session completion when enabled
    - Test that indexer is not called when disabled
    - _Requirements: 4.1_

- [ ] 12. Final checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation
- Property tests validate universal correctness properties using Hypothesis
- Unit tests validate specific examples and edge cases using pytest
- The frontend memory management UI is not included in this plan — it can be a follow-up spec
