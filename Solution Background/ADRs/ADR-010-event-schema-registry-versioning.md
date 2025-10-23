# ADR 10: Event Schema Registry and Versioning Strategy

## Status

Draft

## Context

MobilityCorp's event-driven architecture relies on 50+ event types (vehicles, bookings, deliveries, users, field staff) flowing between 20+ microservices, 4 mobile apps, back office, data warehouse, and external partners. Schemas must evolve without breaking consumers.

**Schema Evolution Challenges**:
- Adding fields: Will old consumers break?
- Removing fields: Will consumers reading them fail?
- Changing types: String→decimal is incompatible
- Renaming fields: Breaking change requiring coordination
- Independent deployments: Service A deploys v2 Monday, Service B still expects v1 until Friday

**Requirements**: Schema validation (producers/consumers validate before publish/consume), version management (track v1/v2/v3), compatibility guarantees (backward/forward/full), code generation (TypeScript, Java, Go), governance (review breaking changes, deprecation policy), audit trail (who changed, when, rollback)

**Regulatory**: GDPR PII documentation in schemas, encryption requirements, retention policies per event type

## Decision

Implement **Confluent Schema Registry** with **Apache Avro** as primary schema format for efficient serialization and strong compatibility guarantees.

**Why Avro**: Compact binary (60-70% smaller than JSON, lower bandwidth), schema evolution built-in (reader vs writer schema), strong typing, cross-language support, self-describing (schema embedded), industry standard (Confluent, LinkedIn, Netflix)

**Compatibility Modes**:
- **Backward** (default): New consumers read old events, add optional fields only, never remove required fields
- **Forward**: Old consumers read new events, never add required fields
- **Full** (recommended for critical): Both backward + forward, most restrictive/safest (booking.created, payment.processed)
- **None**: Breaking changes allowed (experimental/internal events only)

**Versioning**:
- **Semantic**: Major (breaking, v2.0.0), Minor (backward-compatible, v1.1.0), Patch (docs only, v1.0.1)
- **File naming**: vehicle.unlocked.v1.avsc, vehicle.unlocked.v2.avsc
- **Schema Registry subjects**: com.mobilityocorp.events.vehicle.VehicleUnlocked (versions: v1 id:1001, v2 id:1002)

**Schema Evolution Patterns**:
- **Add optional field** (backward compatible): Union type [null, type] with default value
- **Deprecate field** (forward compatible): Make nullable, add replacement field, dual-write 90 days, monitor usage, remove in major version
- **Rename field** (breaking): Dual publishing to v1/v2 topics for 90 days, consumers migrate gradually, stop v1 after migration

**Governance**:
- **Review process**: PR with schema → CI validates (syntax, compatibility check against production) → platform team review → merge → auto-deploy to registry
- **Breaking change policy**: Forbidden without migration plan, 90-day deprecation notice, migration guide, email to consumers, monitor migration progress
- **Deprecation checklist**: Mark deprecated, add replacement, send notice (90 days), reminder (30 days), remove in major version, monitor for errors

**Code Generation**: Auto-generate TypeScript/Java/Go types from Avro schemas in build pipeline, integrate with Maven/Gradle/npm, cache generated code

## Alternatives Considered

**JSON Schema**: Human-readable but larger payloads (text vs binary), no embedded schema, weaker evolution guarantees

**Protobuf**: Similar to Avro but less Kafka ecosystem support, steeper learning curve (use for gRPC services, Avro for Kafka events)

**GraphQL Schema**: Designed for queries not events, overkill

**No Schema Registry** (embed in code): No central validation, schema drift, no compatibility checks, poor governance

**Version in payload** ({"version": "2.0.0", "payload": {}}): Larger payloads, manual validation, no automated compatibility checks

## Consequences

**Positive**:
- Prevent breaking changes (compatibility checks catch before deployment)
- Smooth evolution (backward compatibility allows gradual consumer upgrade)
- Type safety (generated code prevents runtime errors)
- Documentation (schemas are contract, auto-generate API docs)
- Bandwidth savings (Avro binary 60-70% smaller than JSON)
- Audit trail (Schema Registry tracks all changes)
- Developer experience (code gen, IDE autocomplete)

**Negative**:
- Learning curve for Avro syntax and compatibility rules (mitigated by training, docs, templates)
- Schema Registry dependency (critical infrastructure, managed Confluent Cloud or HA setup)
- Binary format harder to debug vs JSON (mitigated by kafka-avro-console-consumer, Schema Registry UI, debug logging)
- Migration complexity for breaking changes (dual publishing, gradual rollout; clear guides)
- Code generation build overhead ~10s (acceptable for type safety, cache generated code)

## Integration Points

- **[ADR-001](./ADR-001-event-driven-architecture-microservices.md) (Event-driven)**: All events validated against Schema Registry
- **[ADR-005](./ADR-005-real-time-vehicle-state-distribution.md) (Real-time state)**: vehicle.* events use Avro schemas
- **[ADR-007](./ADR-007-offline-first-data-synchronization.md) (Offline sync)**: Sync events validated, versioned
- **[ADR-009](./ADR-009-multi-tenancy-geographic-partitioning.md) (Multi-region)**: Schema Registry per region, schemas replicated
