# Hefest — School Events System

Built for **AIBEST 2026 Burgas** by a team of 8.

## What is this?

Hefest is a school event management platform. Students browse and register for published events (lectures, clubs, workshops). Organizers manage the full event lifecycle from draft through to published. When events are full, students join an automatic FIFO waitlist — and are promoted automatically when a confirmed seat is freed. An asynchronous notification pipeline emails students at every meaningful step without slowing down the HTTP API.

## Documentation

| Section | Description |
|---|---|
| [Overview](architecture/index.md) | System topology, technology stack, naming conventions, and repo structure |
| [Data Model](architecture/data-model.md) | Database schema, constraints, and partial indexes |
| [Transactions](architecture/transactions.md) | Registration, cancellation, waitlist promotion, and overbooking prevention |
| [Notification Pipeline](architecture/pipeline.md) | Transactional outbox, LISTEN/NOTIFY relay, domain events, idempotency, and retry |
| [API Reference](architecture/api.md) | Complete REST surface with role enforcement |
| [Operations](architecture/operations.md) | Rate limiting, deployment, scalability, and architecture decisions |
