# ADR 5: Real-time Vehicle State Distribution

## Status

Draft

## Context

MobilityCorp requires real-time vehicle state distribution across customer apps, field staff, on-vehicle screens, and back office systems. Key requirements include:

**Scale**: 10,000 vehicles per city publishing state every 30-60s, 100k+ concurrent users, ~167 updates/second per city

**Consistency needs**:
- Strong consistency: Booking availability (prevent double-booking)
- Eventual consistency: Location (<30s staleness), battery level (<2min staleness)

**Challenges**: Thousands of concurrent clients, mobile battery constraints, network efficiency, geographic filtering (users only see vehicles within 5km), different staleness tolerances per data type

## Decision

Implement **Server-Sent Events (SSE) + Redis** architecture for real-time state distribution with geospatial indexing.

**Architecture**:
1. **Vehicles** publish state to event bus every 30-60s
2. **State Aggregation Service** subscribes to events, updates Redis cache and geospatial index, publishes to Redis pub/sub
3. **Real-time API Service** exposes SSE endpoint, subscribes to Redis pub/sub, filters by geography, streams to clients
4. **Geospatial Query Service** uses Redis GEORADIUS for initial map load and SSE fallback

**Data Flow**: Vehicles → Event Bus → State Aggregator → Redis (cache + geo-index + pub/sub) → SSE API → Client Apps

**Client Strategy**:
- Customer app: SSE when active, close when backgrounded (battery), fallback to 10s polling
- Field staff: SSE when online, periodic polling when unstable, SQLite cache for offline
- On-vehicle: Subscribe only to own state

**Consistency Implementation**:
- Strong consistency operations (booking, unlock, payment) query database directly, not cache
- Eventual consistency (location, battery, fleet overview) use Redis cache with TTL

## Alternatives Considered

**WebSockets**: Rejected due to complexity (connection management, sticky sessions), proxy issues, higher battery drain vs SSE

**GraphQL Subscriptions**: Over-engineered for this use case, doesn't solve core challenges

**Polling**: Poor UX (5-10s latency), inefficient server load

**Managed services** (Firebase, Pusher, Ably): Expensive at scale (100k connections), vendor lock-in, data residency concerns

## Consequences

**Positive**:
- Sub-second latency creates responsive "live" UX
- SSE scales to 100k+ connections, more battery-efficient than WebSockets
- Geographic filtering reduces bandwidth
- Works everywhere (standard HTTP, proxies, firewalls)
- Graceful degradation to polling
- Cost-effective (open-source stack)

**Negative**:
- SSE browser connection limits (mitigated by HTTP/2, dedicated subdomain)
- SSE one-way only (acceptable, use REST for commands)
- Redis single point of failure (mitigated by cluster + replication)
- Eventual consistency may show stale availability (booking API does authoritative check)
- Mobile app connection lifecycle complexity (use robust libraries)

## Integration Points

- **[ADR-001](./ADR-001-event-driven-architecture-microservices.md) (Event-driven)**: Vehicles publish state to event bus, State Aggregator consumes
- **[ADR-002](./ADR-002-separate-apps-for-user-roles.md) (Separate apps)**: Each app streams filtered data (customers: nearby vehicles, field staff: all city)
- **[ADR-003](./ADR-003-vector-store-for-rag-capabilities.md) (Vector Store)**: Location data indexed for "vehicles near landmarks" semantic search
- **[ADR-004](./ADR-004-ai-llm-service-orchestration.md) (AI Service)**: Demand forecasting uses historical state
- **[ADR-007](./ADR-007-offline-first-data-synchronization.md) (Offline sync)**: Field staff caches state locally, syncs via SSE when online
