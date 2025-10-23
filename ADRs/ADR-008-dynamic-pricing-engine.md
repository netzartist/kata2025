# ADR 8: Dynamic Pricing Engine for Revenue Optimization

## Status

Draft

## Context

MobilityCorp needs revenue optimization across vehicle types and user segments (tourists, locals, business, delivery) while maintaining regulatory compliance (no discrimination, surge caps vary by city, transparency required). System must handle real-time calculation (<200ms), price consistency, and explainability.

## Decisions Considered

Implement hybrid rule-based + ML pricing engine. Three layers: transparent rule-based base pricing with time/weather/segment multipliers, ML optimization, and constraint layer enforcing regulatory caps and fairness rules. 

## Alternatives Considered

TODO

## Consequences

**Positive**:
- ML pricing can increase revenue
- Demand management shifts traffic to off-peak

**Negative**:
- Hybrid system complexity (3 layers to maintain)
- ML model maintenance and monitoring overhead

## Integration Points

- **[ADR-001](ADR-001-event-driven-architecture-microservices.md) (Event-driven)**: Pricing events to event bus
- **[ADR-004](ADR-004-ai-llm-service-orchestration.md) (AI Service)**: ML models for demand forecasting and optimization
- **[ADR-005](ADR-005-real-time-vehicle-state-distribution.md) (Real-time state)**: Supply data (e.g. available vehicles) feeds pricing
- **Booking Service**: Price validation at booking time
