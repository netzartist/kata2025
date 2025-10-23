# ADR 8: Dynamic Pricing Engine for Revenue Optimization

## Status

Draft

## Context

MobilityCorp needs to optimize revenue across diverse vehicle types and user segments while maintaining customer trust and regulatory compliance.

**Market Segments**: Tourists (50% bookings, 70% revenue, price insensitive), Locals (30% bookings, 20% revenue, price sensitive), Business travelers (15%/5%), Delivery services (5%/5%)

**Pricing Factors**: Demand (availability, booking velocity), Supply (idle vehicles, battery levels), Temporal (time of day, day of week, seasonality), External (weather, events, transit disruptions, competitors), User-specific (new/regular/subscriber, segments)

**Regulatory Constraints**: No price discrimination on protected characteristics, price transparency required, surge caps (2x Paris, 3x Berlin), VAT compliance, local transport authority maximum rates

**Challenges**: Real-time calculation (<200ms latency), price consistency for same user/vehicle, price lock during booking, 100k calculations/min peak, explainability, A/B testing, model updates without downtime

## Decision

Implement **hybrid rule-based + ML pricing engine** where transparent rules set base price and bounds, ML optimization fine-tunes multiplier within constraints.

**Three-Layer Model**:
1. **Base Pricing (Rules)**: Transparent formula: base_rate + (duration × per_minute_rate) + (distance × per_km_rate), with time/weather/user segment multipliers
2. **ML Optimization**: XGBoost demand forecasting + contextual bandits for price multiplier (0.8x to 2.0x), features include time, location, weather, events, supply, user segment
3. **Constraint Layer**: Enforce regulatory caps, max +30% price increase per hour, loyalty floors, competition ceilings, fairness rules

**Price Lock**: 5-minute guarantee stored in Redis, honored even if price changes during booking flow

**Surge Pricing**: Triggered when supply/demand ratio <0.5, surge_multiplier = min(1.0 + (demand_excess × 0.5), regulatory_max), gradual ramp up/down, transparent notification to users

**Explainability**: Price breakdown shown on request (base + time component + demand + weather + discounts), plain language ("high demand in area" not "1.3x")

**A/B Testing**: Cohort assignment for testing strategies, max 20% price difference between variants, exclude loyal users, monitor conversion rates, kill switch if >10% drop

## Alternatives Considered

**Pure rule-based**: Transparent but doesn't optimize revenue, manual tuning required

**Pure ML**: Optimizes revenue but black-box, regulatory risk, damages trust

**Competitor price matching**: Race to bottom, reactive not proactive, doesn't optimize own supply/demand

**Auction-based** (customers bid): Terrible UX, slow, unfair, regulatory issues

**Fixed subscription-only**: Simple but leaves revenue on table, doesn't manage demand

## Consequences

**Positive**:
- ML pricing increases revenue 15-25% vs fixed pricing
- Demand management shifts traffic to off-peak times/zones
- Rule-based base + explainability builds trust
- Constraints prevent discrimination and excessive surges
- Easy to update rules without retraining models
- Continuous learning from booking behavior

**Negative**:
- Hybrid system complexity (mitigated by layer separation, testing, docs)
- Customer perception of "surge pricing" (mitigated by transparency, alternatives, loyalty discounts)
- ML model maintenance (automated retraining, monitoring)
- Regulatory risk (work with legal, enforce caps, transparency reports)
- Price elasticity risk may reduce bookings (A/B test optimal points, monitor conversion)
- Competitive response (monitor competitors, adjust floors/ceilings)

## Integration Points

- **[ADR-001](./ADR-001-event-driven-architecture-microservices.md) (Event-driven)**: Pricing events (surge.started, price.changed) to event bus
- **[ADR-004](./ADR-004-ai-llm-service-orchestration.md) (AI Service)**: ML models for demand forecasting and optimization
- **[ADR-005](./ADR-005-real-time-vehicle-state-distribution.md) (Real-time state)**: Supply data (available vehicles) feeds pricing
- **Booking Service**: Price validation at booking time
