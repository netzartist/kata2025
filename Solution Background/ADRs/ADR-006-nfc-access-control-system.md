# ADR 6: NFC Access Control System for Vehicle Unlocking

## Status

Draft

## Context

MobilityCorp requires secure vehicle access for multiple user types with different operational needs:

**User Types**:
- **Customers**: Unlock during booking, need offline capability (tunnels, parking garages), expect <2s unlock
- **Delivery Partners (3rd party)**: Temporary single-use access to trunk only, 15-minute window, security critical
- **Field Staff**: Master access to all regional vehicles, 24-hour tokens, must work offline
- **Emergency Override**: Back office manual unlock with approval workflow

**Security Requirements**: Prevent unauthorized access, prevent replay attacks, time-bound tokens, complete audit trail, offline operation for customers/staff, token revocation, no shared secrets per vehicle

**Challenges**: Clock synchronization for time-based tokens, connectivity variability, token lifetime tradeoffs (long=UX, short=security), replay prevention without connectivity, secure key distribution to thousands of vehicles

## Decision

Implement **hybrid cryptographic token system** using JWT with ECDSA signatures, combining server-validated tokens (primary) with time-based offline fallback.

**Token Types**:
1. **Customer Booking**: Hybrid mode (try server, fallback offline), valid for booking duration
2. **Delivery Partner**: Server-only (no offline), single-use, 15-minute validity, trunk access only
3. **Field Staff**: Offline-first, 24-hour validity, wildcard vehicle access in region
4. **Emergency**: Server-only with approval workflow

**Validation Modes**:
- **Server-validated**: Vehicle sends token to backend via cellular, real-time validation/revocation, 3s timeout
- **Time-based offline**: Vehicle validates locally (signature, expiration, local revocation cache), ±5min clock drift tolerance
- **Hybrid**: Attempt server validation, fallback to offline if no connectivity

**Cryptography**:
- JWT tokens with ES256 (ECDSA + SHA-256)
- Each vehicle has unique ECDSA key pair, private key in Hardware Security Module (tamper-resistant)
- Backend signing key rotates every 6 months
- Delivery partner tokens encrypted with vehicle's public key

**Audit Logging**: Every unlock attempt (success/failure) published to event bus with vehicle_id, user_id, token_type, validation_mode, result, GPS location, timestamp

## Alternatives Considered

**Bluetooth Low Energy (BLE)**: Rejected due to longer range enabling range attacks, higher battery drain, requires vehicle battery (NFC works when dead)

**QR Code**: Requires vehicle display screen, poor in dark/weather, slower UX

**PIN Code**: Requires physical keypad (cost, weatherproofing), slow, no user audit trail

**Centralized key server** (always fetch from server): Doesn't work offline, blocks customers without connectivity

## Consequences

**Positive**:
- Cryptographic security prevents unauthorized access
- Offline capability for customers/field staff (time-based fallback)
- Complete audit trail for compliance and disputes
- Flexible permissions per user role
- Immediate revocation for server-validated mode
- Standards-based (JWT, NFC, ECDSA)

**Negative**:
- Clock sync complexity (mitigated by NTP, cellular time, ±5min tolerance, tamper detection)
- Revocation lag in offline mode (mitigated by frequent vehicle sync, short token lifetimes; critical tokens require server)
- Key rotation complexity (mitigated by OTA updates, 30-day dual-key overlap)
- Hardware cost ~$20-50 per vehicle for NFC + HSM (acceptable for security)
- Token size 500-1000 bytes (within NFC limits)

## Integration Points

- **[ADR-001](./ADR-001-event-driven-architecture-microservices.md) (Event-driven)**: All unlock events published to event bus
- **[ADR-002](./ADR-002-separate-apps-for-user-roles.md) (Separate apps)**: Each app generates appropriate token type
- **[ADR-005](./ADR-005-real-time-vehicle-state-distribution.md) (Real-time state)**: Lock state updates distributed after unlock
- **Booking/Delivery Services**: Trigger token generation
