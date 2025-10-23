# ADR 1: Event-driven architecture using microservices

## Status

Decided

## Context

Users want to make vehicle bookings by using their smartphone or on-vehicle screens directly. Therefore they need to see vehicle availability in real-time either on a map or on the vehicle in front of them.

Vehicles send out important real-time information like battery status, geolocation, etc.

## Decision

The backbone of MobilityCorp's system consists of an event-driven architecture using microservices. Offers connectors for the different types of vehicle, user groups (UIs), 3rd party services (APIs), planning systems, etc.

## Consequences

### Performance
React to a big variety of different events in a fast way

### Evolvability
Allows to add and react on more event types in the future as the system grows

### Scalability
Possibility to split and replicate this system for respective regions/cities