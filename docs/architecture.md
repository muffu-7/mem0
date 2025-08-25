## Architecture Overview

This document describes the core architecture, features, and data flows of the repository. It explains how the system implements a comprehensive memory and retrieval platform by combining vector search over unstructured text with knowledge-graph-based extraction, semantic graph search, and LLM-guided maintenance of graph relationships.

### Goals
- Provide a unified memory layer for conversational and factual knowledge.
- Support high-quality retrieval using vector similarity for unstructured content.
- Maintain a structured knowledge graph extracted from conversations for reasoning and long-term coherence.
- Combine vector search and graph search to return rich, complementary context.

## Core Components

### 1) Embedding Layer
- Pluggable embedding providers (OpenAI, Azure OpenAI, HuggingFace, Vertex AI, Gemini, Together, LM Studio, Bedrock, etc.).
- Mode-aware embedding calls (e.g., add, search, update) to support consistent pipelines.
- Used by both the vector store and the graph layer for semantic matching.

### 2) Vector Store Layer
- Unified interface over multiple vector databases (e.g., Qdrant, Pinecone, pgvector, Milvus, FAISS, Weaviate, OpenSearch, Redis, MongoDB, Azure AI Search, Vertex AI Matching Engine, etc.).
- Stores text memories with payload metadata (session identifiers, roles, timestamps, hashes, and custom metadata).
- Supports filtered, scored semantic search over text.

### 3) Graph Store Layer
- Backends for Neo4j, Memgraph, Neptune (Analytics), and Kuzu.
- Stores entities as nodes and relationships as edges with embeddings on nodes.
- LLM-driven extraction of entities and relationships from text, with optional custom prompts.
- Semantic search in the graph to find similar nodes and their neighborhoods.
- LLM-guided deletion of outdated or contradictory edges.
- Upsert logic with semantic attachment and structural deduplication.

### 4) LLM Orchestration
- Provider-agnostic LLM wrapper for:
  - Extracting entities and their types.
  - Establishing relationships among entities.
  - Deciding which graph edges to delete based on new information and current graph context.
  - Extracting concise “facts” for vector storage and later updates.

### 5) Configuration & Factories
- Factories build providers for LLMs, embedders, vector stores, and graph backends from configuration.
- Session scoping and filters are consistently applied across layers (user_id, agent_id, run_id, and optional actor_id).
- Index setup and guardrails per backend to ensure efficient MERGE/upsert behavior.

## Data Model & Indexing

### Vector Memories
- Each memory item holds the original text, an embedding vector, and metadata (including session identifiers and timestamps).
- Vector stores manage similarity indexes appropriate to the provider.

### Graph Entities
- Nodes store name, labels or base label, user scoping identifiers, mentions counters, timestamps, and an embedding vector.
- Relationships store type/name, mentions counters, and temporal metadata.
- Indexes on node identity fields (e.g., user_id, name) and vector indexes for node embeddings when supported.

## Ingestion Flow

1) Input normalization
- Messages are normalized and optionally pre-processed for vision inputs if enabled.

2) Vector-side fact extraction and storage
- The LLM extracts concise facts from messages.
- Each fact is embedded and searched against the vector store to identify similar existing memories for potential update or deletion.
- The LLM decides add/update/delete actions; the store is updated accordingly.

3) Graph-side extraction and maintenance
- The LLM extracts entities and determines relationships.
- Entities are normalized for consistency and safety (e.g., lowercasing, underscore separators, relationship sanitization for graph queries).
- Semantic graph search collects similar nodes and their incoming/outgoing edges as context.
- The LLM decides which relationships to delete based on the new text and the retrieved graph context.
- Upsert with dedup:
  - Semantic node matching (high threshold) attaches relationships to existing nodes when appropriate.
  - Structural dedup via MERGE-equivalent operations ensures no duplicate nodes or edges.
  - Mentions counters and timestamps are updated on matches; embeddings are written or refreshed.

## Retrieval Flow

1) Vector search
- The query is embedded in “search” mode and searched against the vector store.
- Results are filtered by session identifiers and returned with scores and metadata.

2) Graph search
- Entities are extracted from the query.
- Each entity string is embedded and used to find similar nodes by cosine similarity (graph backend specific).
- The system collects incoming and outgoing edges around matched nodes and reranks them for relevance.

3) Combined result
- The system executes vector search and graph search in parallel.
- Returns a unified response that includes:
  - Vector search results (unstructured memories): semantically similar texts.
  - Graph search relations (structured knowledge): relevant triples around entities.
- Clients can fuse these sources for richer LLM prompts, reasoning, and UI.

## Semantic Matching & Deduplication Strategy

- Two thresholds are used to separate retrieval vs. attachment intents:
  - A moderate similarity threshold in graph search to gather contextual neighborhoods for LLM decisions and query-time relations.
  - A higher similarity threshold during upsert to semantically match and re-use existing nodes, preventing duplicates.
- Structural deduplication via MERGE-equivalent upserts on node identity fields and on relationships.
- Embeddings on nodes are maintained or updated to preserve search quality over time.

## Prompting & Tools (Conceptual)

- Entity extraction: prompts instruct the LLM to identify entities and types, including self-references mapped to the current user.
- Relation extraction: prompts encourage consistent, timeless relation names and restrict to relations supported by the current text.
- Deletion decisions: prompts present existing graph memories and new text, with guidance to remove only outdated or contradictory edges.
- Tooling: structured tool calls specify typed payloads for extracted entities, relations, and deletion instructions, ensuring deterministic parsing.

## Concurrency & Performance

- Ingestion runs vector processing and graph processing concurrently per request.
- Search runs vector and graph retrieval concurrently.
- Node/edge indexing, vector indexes, and thresholding keep operations efficient.
- Optional BM25 reranking over graph relation sequences improves relevance for graph outputs.

## Security & Multi-tenant Scoping

- All data operations are scoped by provided session identifiers (user_id, agent_id, run_id) to isolate tenants and sessions.
- Graph queries and vector searches include these filters to prevent cross-tenant leakage.

## Extensibility

- Add new embedding or LLM providers via factories without changing core flows.
- Plug in different vector stores or graph databases.
- Customize prompts for extraction behaviors without altering orchestration.
- Adjust similarity thresholds and indexing strategies per deployment needs.

## Failure Handling & Robustness

- Graceful degradation when structured tool calling is unavailable; non-critical indexes are created opportunistically.
- Defensive parsing around LLM outputs and safe fallbacks for empty results.
- Sanitization of relationship names to avoid backend-specific query issues.

## Summary

The system unifies two complementary retrieval modalities: vector-based similarity over unstructured memories and graph-based reasoning over structured knowledge. By orchestrating LLM-driven extraction/maintenance for the graph and running vector and graph searches in parallel, it provides rich, high-recall, and semantically coherent context for downstream agents and applications. This architecture ensures deduplication, efficient retrieval, and multi-tenant isolation while remaining extensible across infrastructure backends and model providers.