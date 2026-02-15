# Requirements Document

## Introduction

Project Cortex adds a persistent, vector-based semantic memory system to OpenHands. Currently, every agent session starts fresh with no knowledge of user preferences, project-specific patterns, or historical context. Cortex enables the agent to accumulate knowledge across sessions — indexing PRs, code reviews, chat sessions, architectural decisions, and style preferences — and retrieve relevant memories via vector search when starting new tasks. The agent gets smarter over time, reducing repetitive instructions and friction.

## Glossary

- **Cortex**: The semantic memory subsystem that persists and retrieves knowledge across sessions.
- **Memory_Entry**: A single unit of knowledge stored in Cortex, containing text content, an embedding vector, metadata, and a category.
- **Embedding**: A fixed-dimensional numerical vector representation of text, produced by an embedding model, used for similarity search.
- **Embedding_Provider**: A service that converts text into embedding vectors (e.g., OpenAI, local sentence-transformers).
- **Knowledge_Category**: A classification label for a Memory_Entry. Categories include: `style_preference`, `architectural_decision`, `bug_fix`, `code_pattern`, `project_context`, and `workflow`.
- **Memory_Store**: The persistent storage backend that holds Memory_Entry records and their embedding vectors.
- **Vector_Search**: A similarity-based retrieval mechanism that finds Memory_Entry records whose embeddings are closest to a query embedding.
- **Indexer**: The component responsible for extracting knowledge from source materials (PRs, code reviews, chat sessions) and creating Memory_Entry records.
- **Memory_Context**: The set of Memory_Entry records retrieved by Cortex and injected into the agent prompt for a given task.
- **Relevance_Score**: A numerical similarity score (0.0 to 1.0) indicating how closely a Memory_Entry matches a query.

## Requirements

### Requirement 1: Store Knowledge Entries

**User Story:** As an OpenHands user, I want the system to persistently store knowledge entries so that context accumulates across sessions.

#### Acceptance Criteria

1. WHEN a Memory_Entry is created with text content, a Knowledge_Category, and metadata, THE Memory_Store SHALL persist the Memory_Entry with a unique identifier, a timestamp, and an embedding vector.
2. WHEN a Memory_Entry is persisted, THE Memory_Store SHALL store the entry so that the entry is retrievable by its unique identifier.
3. WHEN a Memory_Entry with a duplicate unique identifier is stored, THE Memory_Store SHALL reject the duplicate and return an error.
4. IF the Embedding_Provider is unavailable, THEN THE Memory_Store SHALL queue the Memory_Entry for later embedding generation and store it without an embedding vector.
5. WHEN a Memory_Entry is deleted by its unique identifier, THE Memory_Store SHALL remove the entry and its embedding vector from persistent storage.

### Requirement 2: Generate Embeddings

**User Story:** As an OpenHands user, I want text content to be automatically converted into embedding vectors so that semantic search is possible.

#### Acceptance Criteria

1. WHEN text content is provided to the Embedding_Provider, THE Embedding_Provider SHALL return a fixed-dimensional embedding vector.
2. WHEN the same text content is provided to the Embedding_Provider multiple times, THE Embedding_Provider SHALL return identical embedding vectors (deterministic output).
3. IF the text content exceeds the Embedding_Provider token limit, THEN THE Embedding_Provider SHALL truncate the text to the maximum allowed length and generate an embedding for the truncated text.
4. IF the Embedding_Provider encounters a network or service error, THEN THE Embedding_Provider SHALL raise a descriptive error with the failure reason.
5. THE Embedding_Provider SHALL support configuration of the embedding model name and dimensionality through the OpenHands configuration system.

### Requirement 3: Retrieve Relevant Memories via Vector Search

**User Story:** As an OpenHands user, I want the agent to retrieve relevant past knowledge when starting a new task so that I do not need to repeat context.

#### Acceptance Criteria

1. WHEN a query string is provided to Cortex, THE Vector_Search SHALL return Memory_Entry records ordered by descending Relevance_Score.
2. WHEN a query is performed with a maximum result count parameter, THE Vector_Search SHALL return at most that number of Memory_Entry records.
3. WHEN a query is performed with a minimum Relevance_Score threshold, THE Vector_Search SHALL exclude Memory_Entry records below that threshold.
4. WHEN a query is performed with a Knowledge_Category filter, THE Vector_Search SHALL return only Memory_Entry records matching the specified category.
5. WHEN no Memory_Entry records match the query above the threshold, THE Vector_Search SHALL return an empty result set.

### Requirement 4: Index Source Materials

**User Story:** As an OpenHands user, I want the system to automatically extract knowledge from my past interactions so that the agent learns from my history.

#### Acceptance Criteria

1. WHEN a conversation session completes, THE Indexer SHALL extract key decisions, preferences, and patterns from the session and create corresponding Memory_Entry records.
2. WHEN indexing a conversation session, THE Indexer SHALL assign an appropriate Knowledge_Category to each extracted Memory_Entry.
3. WHEN indexing content that duplicates an existing Memory_Entry (by semantic similarity above a configurable threshold), THE Indexer SHALL update the existing entry timestamp rather than create a duplicate.
4. IF the Indexer encounters unparseable content, THEN THE Indexer SHALL log a warning and skip the unparseable segment without failing the entire indexing operation.

### Requirement 5: Inject Memory Context into Agent Prompts

**User Story:** As an OpenHands user, I want the agent to proactively use relevant memories when working on my tasks so that the agent behaves consistently with my preferences.

#### Acceptance Criteria

1. WHEN a new agent task begins, THE Cortex SHALL query for relevant Memory_Entry records using the task description as the query string.
2. WHEN relevant Memory_Entry records are found, THE Cortex SHALL format the Memory_Context as a structured text block and inject it into the agent system prompt.
3. WHEN the total size of the Memory_Context exceeds a configurable token budget, THE Cortex SHALL truncate the Memory_Context by removing the lowest-scoring Memory_Entry records until the budget is met.
4. WHEN no relevant Memory_Entry records are found, THE Cortex SHALL proceed without injecting any Memory_Context into the prompt.

### Requirement 6: Serialize and Deserialize Memory Entries

**User Story:** As a developer, I want Memory_Entry records to be serializable to JSON so that they can be stored, transferred, and backed up.

#### Acceptance Criteria

1. THE Memory_Store SHALL serialize Memory_Entry records to JSON format for persistence.
2. THE Memory_Store SHALL deserialize JSON data back into Memory_Entry records.
3. FOR ALL valid Memory_Entry records, serializing then deserializing SHALL produce an equivalent Memory_Entry (round-trip property).

### Requirement 7: Configure Cortex via OpenHands Configuration

**User Story:** As an OpenHands administrator, I want to configure Cortex settings through the existing configuration system so that deployment is consistent.

#### Acceptance Criteria

1. THE Cortex SHALL read configuration values from the OpenHands configuration system, including embedding model name, vector dimensionality, default relevance threshold, maximum memory context token budget, and storage backend path.
2. WHEN configuration values are missing, THE Cortex SHALL use sensible default values and log the defaults being used.
3. WHEN invalid configuration values are provided (e.g., negative dimensionality), THE Cortex SHALL raise a validation error with a descriptive message at startup.

### Requirement 8: Manage Memory Entries via API

**User Story:** As an OpenHands user, I want to view, search, and delete my stored memories through the frontend so that I have control over what the agent remembers.

#### Acceptance Criteria

1. WHEN a user requests a list of Memory_Entry records, THE Cortex API SHALL return a paginated list of entries with their metadata, category, and relevance information.
2. WHEN a user searches Memory_Entry records with a text query, THE Cortex API SHALL return matching entries ordered by Relevance_Score.
3. WHEN a user deletes a Memory_Entry by identifier, THE Cortex API SHALL remove the entry from the Memory_Store and confirm deletion.
4. WHEN a user requests Memory_Entry records filtered by Knowledge_Category, THE Cortex API SHALL return only entries matching the specified category.
