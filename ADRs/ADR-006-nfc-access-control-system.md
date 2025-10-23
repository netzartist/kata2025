# ADR 6: NFC Access Control System for Vehicle Unlocking

## Status

Draft

## Context

MobilityCorp requires secure vehicle access for customers, delivery partners, field staff, and emergency overrides. Key challenges include offline operation requirements, preventing unauthorized access/replay attacks, clock synchronization for time-based tokens, and secure key distribution across thousands of vehicles.

## Decisions Considered

Have different token types to manage authorization and allow NFC access

## Alternatives Considered

TODO

## Consequences

TODO

## Integration Points

- **[ADR-001](ADR-001-event-driven-architecture-microservices.md) (Event-driven)**: All unlock events published to event bus
- **[ADR-002](ADR-002-separate-apps-for-user-roles.md) (Separate apps)**: Each app generates appropriate token type
- **[ADR-005](ADR-005-real-time-vehicle-state-distribution.md) (Real-time state)**: Lock state updates distributed after unlock
- **Booking/Delivery Services**: Trigger token generation
