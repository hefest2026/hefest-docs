# Grading Criteria

**Competition:** AIBEST 2026 Burgas — School Events & Notification Center
**Total: 170 points**

Each criterion is assigned a short code used throughout the architecture spec to annotate which criteria each design decision and feature addresses.

---

## Criteria codes

| Code | Category | Points | Criterion summary |
|------|----------|-------:|-------------------|
| **F1** | Functionality | 10 | Authentication & role-based access control |
| **F2** | Functionality | 10 | Event management (CRUD, publish, cancel) |
| **F3** | Functionality | 15 | Registration core (confirmed/waitlist, no overbooking, cancellation, automatic promotion) |
| **F4** | Functionality | 5 | Organizer visibility (registrations + ordered waitlist for own events) |
| **B1** | Backend | 10 | Event-driven processing — separate worker, producer/consumer clearly separated |
| **B2** | Backend | 10 | Database — schema, migrations, constraints, uniqueness |
| **B3** | Backend | 10 | RESTful API — resources, methods, status codes, auth on protected routes |
| **A1** | Additional | 10 | Domain events — ≥ 3 distinct types end-to-end through the worker |
| **A2** | Additional | 5 | Real email delivery + persistent notification log proving delivery attempts |
| **A3** | Additional | 5 | Reliability — job states, retries / max-attempt policy, failures visible |
| **A4** | Additional | 4 | Idempotency — retries must not produce duplicate user-visible notifications |
| **R1** | REST endpoints | 10 | Complete & functional implementation of the core REST surface |
| **S1** | Security | 5 | Secure password storage using slow hashing |
| **S2** | Security | 5 | Input validation + safe query patterns (SQL injection, XSS) |
| **S3** | Security | 5 | Authorization — role and ownership checks on every sensitive operation |
| **SC1** | Scalability | 5 | Design considerations for large user/item counts |
| **SC2** | Scalability | 5 | Database optimization for performance and scalability |
| **SC3** | Scalability | 5 | Architecture scalability for increased traffic |
| **CQ1** | Code quality | 3 | Clean, organized project layout (API vs worker, config, migrations) |
| **CQ2** | Code quality | 3 | Consistent naming and formatting |
| **CQ3** | Code quality | 2 | Comments / short docs where logic is non-obvious |
| **CQ4** | Code quality | 2 | Modular structure and reuse (shared types/DTOs, shared DB access) |
| **P1** | Presentation | 5 | Clarity and coherence of the presentation (problem, solution, architecture) |
| **P2** | Presentation | 5 | Live demonstration of key flows |

---

## Full criteria text

### Functionality (40 points)

| Code | Criterion | Points |
|------|-----------|-------:|
| **F1** | User authentication and role-based access control (student vs organizer): registration/login as documented; 403 when a role tries forbidden actions; students cannot see other users' registrations. | 10 |
| **F2** | Events: organizers can create/update draft events; publish (DRAFT → PUBLISHED); cancel or close an event with documented behavior for existing registrations; students only see/list published events; event ownership (`organizer_id`) enforced. | 10 |
| **F3** | Registration core: register → CONFIRMED if capacity available, else WAITLISTED with FIFO ordering; no overbooking of confirmed seats under concurrency (locking, transaction, or equivalent); student can cancel own registration per team's documented rule; automatic waitlist promotion when a confirmed seat is freed. | 15 |
| **F4** | Organizer visibility: list confirmed registrations and ordered waitlist for own events only; reasonable event detail for capacity/waitlist counts. | 5 |

### Backend (30 points)

| Code | Criterion | Points |
|------|-----------|-------:|
| **B1** | Event-driven processing: domain changes enqueue work; separate worker process (not the HTTP server thread) consumes a queue (DB outbox, Redis, RabbitMQ, etc.); producer/consumer clearly separated in repo and README. | 10 |
| **B2** | Database: relational schema for users, events, registrations, notification jobs; migrations; constraints/uniqueness (e.g. one active registration per user per event). | 10 |
| **B3** | RESTful API: resources and HTTP methods match requirements (auth, events, publish/cancel, registrations, registrations/me, organizer lists); consistent status codes and error shape; authentication on protected routes. | 10 |

### Additional features (25 points)

| Code | Criterion | Points |
|------|-----------|-------:|
| **A1** | Domain events: at least three distinct queued job/event type values (e.g. `RegistrationConfirmed`, `RegistrationWaitlisted`, `WaitlistPromoted`) end-to-end through the worker. | 10 |
| **A2** | Notifications: real email (third-party or SMTP) and persistent notification log (DB table and/or structured console) proving delivery attempts. | 5 |
| **A3** | Reliability: job states (`pending / processing / sent / failed` or equivalent); retries or clear max-attempt policy; failures visible in logs. | 5 |
| **A4** | Idempotency: clear strategy so retries do not duplicate user-visible notifications (unique key, dedupe table, or equivalent). | 4 |

### REST endpoints (10 points)

| Code | Criterion | Points |
|------|-----------|-------:|
| **R1** | Complete and functional implementation of the core REST surface (auth, events including publish/cancel, registrations including cancel and "me", organizer registration/waitlist reads); behavior must match the spec's intent. | 10 |

### Security (15 points)

| Code | Criterion | Points |
|------|-----------|-------:|
| **S1** | Passwords: secure storage using slow hashing — not reversible "encryption". | 5 |
| **S2** | Input validation and safe query/data access patterns to reduce SQL injection and XSS (if any HTML is rendered). | 5 |
| **S3** | Authorization: every sensitive read/write checks role and ownership (event organizer, registration owner); mechanisms to control access to sensitive data. | 5 |

### Scalability & design (15 points)

| Code | Criterion | Points |
|------|-----------|-------:|
| **SC1** | Design considerations for scalability to handle a large number of users and items. | 5 |
| **SC2** | Optimization of the database for performance and scalability. | 5 |
| **SC3** | Scalability of the project architecture to handle increased traffic and demand. | 5 |

### Code quality (10 points)

| Code | Criterion | Points |
|------|-----------|-------:|
| **CQ1** | Clean, organized project layout (API vs worker, config, migrations). | 3 |
| **CQ2** | Consistent naming and formatting (formatter/linter optional but welcome). | 3 |
| **CQ3** | Comments or short docs where logic is non-obvious (e.g. waitlist transaction). | 2 |
| **CQ4** | Modular structure and reuse (shared types/DTOs, shared DB access patterns). | 2 |

### Presentation (10 points)

| Code | Criterion | Points |
|------|-----------|-------:|
| **P1** | Clarity and coherence of the project presentation (problem, solution, architecture). | 5 |
| **P2** | Demonstration of key flows: publish → register → waitlist → cancel → promotion → async notification visible. | 5 |
