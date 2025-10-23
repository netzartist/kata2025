# ADR 9: Multi-Tenancy and Geographic Partitioning Strategy

## Status

Draft

## Context

MobilityCorp is expanding from single-city (Berlin) to multi-city, multi-country platform with complex regulatory and operational requirements.

**Expansion Plan**: Phase 1 (Berlin 10k vehicles), Phase 2 (EU: Paris, Barcelona, Amsterdam - 40k vehicles), Phase 3 (Global: NYC, SF, Singapore - 70k vehicles)

**City-Specific Variations**:
- **Regulatory**: France max 2x surge + data localization, Germany GDPR, USA state-specific (CCPA), Singapore PDPA
- **Operational**: Vehicle types differ (Paris bans escooters in districts), delivery partners vary, payment methods (SEPA/ACH/local), languages
- **Pricing**: Base rates vary, competitive landscapes differ, currencies/tax

**Data Residency**: GDPR (EU PII in EU, cross-border needs SCCs, 4% revenue fines), CCPA (less strict), China (future: strict localization), Singapore (consent required)

**Cross-Region Scenarios**: Tourist books in multiple cities (Berlin→Paris), corporate accounts across cities, field staff work in multiple cities, delivery partners operate multi-city

**Scaling**: Independent per-city scaling, peak hours vary by timezone, seasonal patterns differ, event-driven spikes localized

## Decision

Implement **regional cluster architecture** with logical city partitioning, prioritizing data residency compliance while enabling cross-region functionality.

**Regional Deployment**:
- **EU Region** (Frankfurt): Kubernetes cluster with Berlin/Paris/Barcelona/Amsterdam namespaces, shared event bus/AI/auth
- **US Region** (N. Virginia): NYC/SF namespaces, regional shared services
- **Asia Region** (Singapore): Singapore namespace, regional services
- **Global Services**: Auth (multi-region), Payment (Stripe external), CDN, Data Warehouse (centralized)

**Database Architecture**:
- Database per city (berlin_db, paris_db, etc.) in regional PostgreSQL clusters
- Logical isolation prevents cross-city leakage, independent scaling
- Global user table in Auth Service, regional profiles in city databases

**Multi-Region User Management**:
- Global user ID with regional profiles
- Cross-region replication via async event bus (user.created, user.updated)
- Tourist travels Berlin→NYC: US region replicates user profile from EU (consent-based, minimal PII)
- Billing aggregates across regions

**Service Discovery**: DNS-based routing (api.mobilityocorp.com → eu/us/asia endpoints), mobile app detects GPS location and routes to nearest region

**Data Residency Compliance**:
- EU customer PII stays in EU region (Frankfurt/Ireland)
- Cross-border transfers only with SCCs and user consent
- Audit log all cross-region replications

**Operational**: Kubernetes namespaces per city with CPU/memory quotas, separate node pools, network policies isolate namespaces, shared regional services (event bus, AI, auth, monitoring)

## Alternatives Considered

**Single global database**: GDPR violations, blast radius affects all cities, scaling bottleneck

**Fully isolated per-city**: High cost (3x infrastructure), complex global features, operational overhead

**Microservices per city** (separate clusters): Extremely high cost, complexity not justified

**Multi-tenant DB** (city_id column): Data residency violations, row-level security complexity, performance bottleneck

## Consequences

**Positive**:
- GDPR compliance (PII stays in region)
- Independent city scaling (NYC peak doesn't affect Berlin)
- Blast radius containment (Paris outage doesn't affect Berlin)
- Regional routing reduces latency
- Regulatory flexibility (adapt to local laws)
- Cost-effective (shared regional infrastructure vs fully isolated)
- Seamless user travel (cross-region replication)

**Negative**:
- Multi-region complexity (mitigated by IaC, docs, runbooks)
- Cross-region latency for user travel (<2s first booking in new region, acceptable, rare)
- Data consistency challenges (eventual consistency acceptable, conflict resolution rules)
- Operational overhead (manage multiple clusters, monitoring, deployments via GitOps)
- Cost (cross-region replication, multiple DBs, regional infrastructure; selective replication, caching)
- Testing complexity (staging per region, automated integration tests)

## Integration Points

- **[ADR-001](./ADR-001-event-driven-architecture-microservices.md) (Event-driven)**: Event bus per region, selective cross-region replication
- **[ADR-002](./ADR-002-separate-apps-for-user-roles.md) (Separate apps)**: Apps auto-detect region, route to nearest
- **[ADR-005](./ADR-005-real-time-vehicle-state-distribution.md) (Real-time state)**: Redis per city, no cross-region sync
- **[ADR-010](./ADR-010-event-schema-registry-versioning.md) (Schema registry)**: Shared per region
