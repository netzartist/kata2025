# ADR 7: Offline-First Data Synchronization for Field Staff

## Status

Draft

## Context

Field staff work in areas with bad connectivity (underground parking, tunnels, suburban areas) executing 20+ tasks per shift.

## Decisions Considered

Implement Event Sourcing with local event store. Events eventually published to event bus after successful sync.

## Alternatives Considered

TODO

## Consequences

**Positive**:
- Supports offline operation for field staff
- Instant UI updates without network latency
- Deterministic conflict resolution (automated)

**Negative**:
- Event sourcing complexity vs CRUD
- Conflict resolution rules require careful design

## Integration Points

- **[ADR-001](ADR-001-event-driven-architecture-microservices.md) (Event-driven)**: Local events eventually published to event bus after sync
- **[ADR-002](ADR-002-separate-apps-for-user-roles.md) (Separate apps)**: Field staff app offline-first, others online-first
- **[ADR-005](ADR-005-real-time-vehicle-state-distribution.md) (Real-time state)**: Vehicle updates from field staff flow through sync to real-time API
- **[ADR-006](ADR-006-nfc-access-control-system.md) (NFC access)**: Unlock events logged locally, synced for audit
