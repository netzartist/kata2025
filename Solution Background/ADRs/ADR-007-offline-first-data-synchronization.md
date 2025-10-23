# ADR 7: Offline-First Data Synchronization for Field Staff

## Status

Draft

## Context

Field staff perform battery swaps, vehicle redistribution, maintenance, and delivery coordination in challenging connectivity environments (underground parking, tunnels, suburban areas with 30-50% connectivity during shifts). They work 8-hour shifts executing 50-100 tasks/day with 500 staff per city = 25k-50k daily operations.

**Requirements**: Full offline operation for entire shift, persistent across app restarts, eventual consistency acceptable, deterministic conflict resolution when multiple staff work on same vehicle, complete audit trail, resume interrupted syncs

**Conflict Scenarios**:
- Duplicate battery swap: Both staff credited, most recent battery level wins
- Vehicle redistribution race: First timestamp wins, second notified
- Delivery task completion: First wins, duplicate flagged
- Offline extended period: Server task reassignments synced on return, local tasks marked cancelled

## Decision

Implement **Event Sourcing with local SQLite event store** for offline-first synchronization, providing deterministic conflict resolution and complete audit trail aligned with event-driven architecture.

**Architecture**:
- **Local Event Store** (SQLite): Immutable event log with sync status tracking
- **State Cache** (SQLite): Materialized views (vehicles, tasks, deliveries) for fast queries
- **Sync Engine**: Upload/download events with priority tiers, handle conflicts
- **Vector Clock Manager**: Track causality across devices for conflict detection
- **Conflict Resolver**: Deterministic rules (battery swap, redistribution, task completion)

**Event Types**: vehicle.battery.swapped, vehicle.location.updated, vehicle.damage.reported, task.completed, delivery.completed, etc.

**Sync Priority Tiers**:
1. **Critical** (30s SLA): battery.swapped, task.completed, delivery.completed
2. **High** (5min SLA): vehicle.damage.reported, task.started, delivery.in_transit
3. **Low** (1hr SLA): vehicle.cleaned, field notes, photos

**Sync Protocol**:
1. Prepare: Query pending events, sort by priority/timestamp, batch 100 events
2. Upload: POST to /sync/events/upload, server validates and accepts/rejects
3. Download: Fetch server events since last sync
4. Apply: Insert server events to local store, update vector clock, materialize state
5. Mark synced: Update sync_status for accepted events

**Conflict Resolution Rules**:
- Battery swap conflicts: Accept both, most recent battery level wins, flag for review
- Redistribution conflicts: First timestamp wins, later rejected
- Task completion conflicts: First wins if >60s apart, both credited if within 60s (race condition)

**Data Retention**: Events 30 days local (purge after sync), tasks 7 days, vehicle cache all assigned vehicles

## Alternatives Considered

**CRDTs**: Rejected due to complexity, larger payloads, overkill for field staff workflows

**Operational Transformation (OT)**: Designed for text editing not task management, very complex

**Last-Write-Wins**: Rejected due to data loss on conflicts (field staff must get credit for work)

**Offline-First DBs** (PouchDB, Couchbase Lite): Vendor lock-in, limited query capabilities vs SQLite, event sourcing better fit

**Firebase Firestore**: Vendor lock-in, expensive at scale, limited conflict resolution control, GDPR concerns

## Consequences

**Positive**:
- Complete offline operation (entire 8-hour shift)
- Instant UI updates (no network latency)
- Deterministic conflict resolution (automated, no manual intervention)
- Complete audit trail (immutable events for compliance)
- Resilient to failures (app crashes, network interruptions)
- Natural fit with event-driven architecture
- Eventual consistency converges when synced
- Bandwidth efficient (delta sync)

**Negative**:
- Event sourcing complexity vs CRUD (mitigated by reusable library, testing, documentation)
- Storage overhead: events + state (~15MB typical, 200MB max, acceptable)
- Conflict resolution rules require careful design (mitigated by clear docs, A/B test with field staff)
- Vector clock management complexity (encapsulated in library)
- Debugging requires event log replay (built-in event viewer, export logs)
- Initial sync latency for 1000s of tasks (background sync, progress bar, aggressive caching)

## Integration Points

- **[ADR-001](./ADR-001-event-driven-architecture-microservices.md) (Event-driven)**: Local events eventually published to event bus after sync
- **[ADR-002](./ADR-002-separate-apps-for-user-roles.md) (Separate apps)**: Field staff app offline-first, others online-first
- **[ADR-004](./ADR-004-ai-llm-service-orchestration.md) (AI Service)**: Route optimization runs on local offline model
- **[ADR-005](./ADR-005-real-time-vehicle-state-distribution.md) (Real-time state)**: Vehicle updates from field staff flow through sync to real-time API
- **[ADR-006](./ADR-006-nfc-access-control-system.md) (NFC access)**: Unlock events logged locally, synced for audit
