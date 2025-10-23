# ADR 2: Have separate Apps: Customers/ Field Staff/ Back Office

## Status

Decided

## Context

MobilityCorp provides short-term vehicle rental for last-mile transport (scooters, eBikes, cars, vans) with integrated delivery services targeting tourists and travelers. Partners with AirBnB, Booking.com, and local food businesses to offer seamless mobility + delivery experiences (e.g., rental car with "welcome package" in trunk, food delivery to parked vehicles).

Different personas have fundamentally different operational needs:

- **Customers (Tourists & Locals)**: Mobile app for vehicle booking, NFC unlock, delivery ordering with AI chat (natural language food requests), per-minute payment, photo-based return. Need AI-powered availability predictions, personalized recommendations, and delivery coordination.

- **Field Staff**: Work in vans with spotty connectivity doing battery swaps, vehicle redistribution, and delivery coordination. Need offline-first mobile app with AI route optimization, battery prioritization, and delivery task management.

- **Delivery Partners (3rd party)**: Need limited access to vehicles via NFC for secure food delivery (e.g., placing orders in trunk while tourist is at museum).

- **On-Vehicle Screens**: Kiosk mode for walk-up bookings on cars/vans. Public access, embedded hardware, must work offline.

- **Back Office**: Desktop web app for fleet management, customer support, analytics, delivery operations, and partner management across all locations. Need comprehensive AI features (demand forecasting, dynamic pricing, churn prediction, delivery routing).

## Decision

Build 4 separate applications optimized for different user types and platform requirements:

1. **Customer App**: Native mobile (iOS/Android) for vehicle booking, NFC unlock, payments, AI-powered delivery ordering
2. **Field Staff App**: Offline-first mobile for battery swaps, vehicle redistribution, and delivery logistics
3. **On-Vehicle App**: Embedded kiosk for walk-up bookings (cars/vans only)
4. **Back Office App**: Desktop web for fleet management, support, analytics, delivery operations, partner integration

**Note**: Delivery partners use limited API access (no dedicated app) with temporary NFC tokens for vehicle access.

**Key reasons for separation**:
- **Offline requirements**: Field staff need full offline mode; customers need minimal offline; on-vehicle needs offline fallback
- **Platform optimization**: Mobile consumer vs mobile offline-first vs embedded kiosk vs desktop
- **Security**: Public kiosk vs customer payment data vs field operations vs admin controls require different scopes
- **AI deployment**: Lightweight mobile inference (customer) vs offline route optimization (field) vs complex ML models (back office)

Microservices publish events to event bus; apps subscribe to relevant streams. BFF pattern optimizes data delivery per app.

## Alternatives Considered

**Single unified app with role-based UI**: Rejected because field staff offline-first requirements conflict with customer real-time needs, and kiosk security model is incompatible with authenticated mobile apps. Would result in bloated customer app and poor field staff experience.

## Consequences

**Positive**:
- Each app optimized for its platform (mobile consumer, offline-first field, kiosk, desktop)
- Better security: Limited scopes per app, customer app doesn't contain admin code
- Independent deployment: Update customer app without affecting field operations
- AI optimization: Mobile inference (customer) vs complex server-side models (back office)

**Negative**:
- 4 codebases and CI/CD pipelines to maintain
- Integration testing spans multiple apps
- Shared business logic must be in backend to avoid duplication
- ML model versioning across apps requires coordination

## Technical Implications

- Event bus for microservices with fan-out to multiple apps (vehicle state, bookings, deliveries, field operations)
- BFF (Backend for Frontend) pattern: Each app gets optimized data delivery and AI inference
- OAuth2 with app-specific scopes (customer, field staff, delivery partners, back office have different permissions)
- Centralized ML platform with versioned models for consistent AI across apps (including NLP for delivery orders)
- Strong consistency for bookings/payments/deliveries; eventual consistency for vehicle location/battery levels
- API for delivery partners: Temporary NFC token generation, vehicle access logs, delivery status updates
