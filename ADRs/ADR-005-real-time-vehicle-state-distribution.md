# ADR 5: Real-time Vehicle State Distribution

## Status

Draft

## Context

MobilityCorp requires real-time vehicle state distribution to customer apps, field staff, and back office. System must handle 10k vehicles per city, 100k+ concurrent users, with sub-second latency for location updates while managing different consistency requirements (strong for bookings, eventual for location/battery).

## Decisions Considered

Implement Server-Sent Events (SSE) with Redis for state caching and geospatial indexing. Vehicles publish to event bus, State Aggregation Service updates Redis cache/geo-index and publishes to Redis pub/sub, Real-time API streams filtered updates to clients via SSE. Strong consistency operations query database directly, eventual consistency uses Redis cache.

## Alternatives Considered

TODO

## Consequences

**Positive**:
- Sub-second latency creates responsive "live" UX
- SSE scales and is battery-efficient
- Geographic filtering reduces bandwidth

**Negative**:
- SSE browser connection limits (mitigated by HTTP/2)
- Eventual consistency may show stale availability

## Integration Points

- **[ADR-001](ADR-001-event-driven-architecture-microservices.md) (Event-driven)**: Vehicles publish state to event bus, State Aggregator consumes
- **[ADR-002](ADR-002-separate-apps-for-user-roles.md) (Separate apps)**: Each app streams filtered data (customers: nearby vehicles, field staff: all city)
- **[ADR-003](ADR-003-vector-store-for-rag-capabilities.md) (Vector Store)**: Location data indexed for "vehicles near landmarks" semantic search
- **[ADR-004](ADR-004-ai-llm-service-orchestration.md) (AI Service)**: Demand forecasting uses historical state
- **[ADR-007](ADR-007-offline-first-data-synchronization.md) (Offline sync)**: Field staff caches state locally, syncs via SSE when online
