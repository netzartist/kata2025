# ADR 3: Vector Store Service for RAG Capabilities

## Status

Decided

## Context

MobilityCorp's AI-powered features require understanding unstructured data sources to provide intelligent responses:

- **Natural language food requests**: Customers use AI chat to order deliveries ("authentic local cuisine, vegetarian, serves 2" or "Welcome to Berlin local breakfast items")
- **Location-based recommendations**: Need to find relevant restaurants, delivery partners, and local businesses based on semantic similarity
- **Customer support**: AI-powered support needs to search historical issues, FAQs, and resolution patterns
- **Delivery partner matching**: Match delivery requests with suitable partners based on capabilities, location, and specialties

Traditional keyword search is insufficient for semantic understanding. A vector database enables finding relevant information even when exact terms don't match.

## Decision

Provide an internal Vector Store Service using **Qdrant**, which is a proven, capable, and high-performing vector database system.

**Key capabilities**:
- Store embeddings for structured data (rides, locations, events, restaurants, delivery partners)
- Support hybrid search combining vector similarity with metadata filtering (city, region, category)
- Enable semantic search for natural language queries
- Provide low-latency retrieval for real-time AI inference

**Architecture**:
- Fed by data pipelines that convert structured data into embeddings
- Indexed alongside metadata (city, region, business type) for hybrid search
- Exposed as internal microservice with event-driven updates
- Integrated with AI/LLM services for Retrieval-Augmented Generation (RAG)

## Alternatives Considered

**PostgreSQL with pgvector**: Rejected because it's optimized for relational data, not purpose-built for vector operations. Performance degrades at scale.

**Pinecone (managed service)**: Rejected to avoid vendor lock-in and external dependencies. We need full control for data privacy and offline development.

**Build custom vector search**: Rejected due to complexity and maintenance burden. Qdrant provides battle-tested algorithms and optimizations.

## Consequences

**Positive**:
- **Semantic understanding**: Enables finding relevant information in unstructured data sources for agentic use cases
- **Improved AI responses**: RAG provides context-aware answers grounded in actual business data
- **Scalability**: Qdrant handles millions of vectors with low-latency retrieval
- **Flexible filtering**: Hybrid search combines semantic similarity with business logic (location, categories)
- **Real-time updates**: Event-driven pipeline keeps embeddings current with operational data

**Negative**:
- **Vendor risk**: Exposure to particular vector store poses risk if project is not maintained
  - *Mitigation*: Replacing a vector store is expected to be a rather isolated change. Abstraction layer limits coupling.
- **Embedding model dependency**: Changing embedding models requires re-indexing all data
  - *Mitigation*: Version embeddings and support gradual migration
- **Operational complexity**: Additional service to monitor, scale, and maintain
- **Cost**: Storage and compute for embeddings, though offset by improved customer experience

## Technical Implications

- **Data pipelines**: Event-driven embeddings generation from structured data changes
- **Embedding models**: Centralized ML service for consistent vector generation across use cases
- **API design**: Abstraction layer to decouple apps from specific vector store implementation
- **Monitoring**: Track search latency, index size, and retrieval relevance metrics
- **Backup/recovery**: Regular snapshots of vector indices alongside metadata
