# API Reference

> **Covers:** [B3](../criteria/grading-criteria.md#backend-30-points) · [R1](../criteria/grading-criteria.md#rest-endpoints-10-points) · [S3](../criteria/grading-criteria.md#security-15-points) — complete REST surface with role enforcement.

All protected routes require `Authorization: Bearer <token>`.

---

## Authentication

| Method | Path | Role | Description | Criteria |
|---|---|---|---|---|
| `POST` | `/register` | Public | Create student account (organizers via seed script — see README) | [F1](../criteria/grading-criteria.md#functionality-40-points) · [S1](../criteria/grading-criteria.md#security-15-points) · [S2](../criteria/grading-criteria.md#security-15-points) |
| `POST` | `/login` | Public | Returns JWT | [F1](../criteria/grading-criteria.md#functionality-40-points) · [S1](../criteria/grading-criteria.md#security-15-points) |

---

## Events

| Method | Path | Role | Description | Criteria |
|---|---|---|---|---|
| `POST` | `/events` | Organizer | Create event in DRAFT | [F2](../criteria/grading-criteria.md#functionality-40-points) · [S3](../criteria/grading-criteria.md#security-15-points) |
| `GET` | `/events` | Both | Students: published only. Organizers: own drafts + published | [F2](../criteria/grading-criteria.md#functionality-40-points) · [S3](../criteria/grading-criteria.md#security-15-points) |
| `GET` | `/events/{id}` | Both | Event details + confirmed count + capacity + waitlist size | [F2](../criteria/grading-criteria.md#functionality-40-points) · [F4](../criteria/grading-criteria.md#functionality-40-points) |
| `PUT` | `/events/{id}` | Organizer (owner) | Edit event (only if still in DRAFT) | [F2](../criteria/grading-criteria.md#functionality-40-points) · [S3](../criteria/grading-criteria.md#security-15-points) |
| `POST` | `/events/{id}/publish` | Organizer (owner) | DRAFT → PUBLISHED | [F2](../criteria/grading-criteria.md#functionality-40-points) · [S3](../criteria/grading-criteria.md#security-15-points) |
| `POST` | `/events/{id}/cancel` | Organizer (owner) | Cancel event; document behavior for existing registrations | [F2](../criteria/grading-criteria.md#functionality-40-points) · [S3](../criteria/grading-criteria.md#security-15-points) |

---

## Registrations

| Method | Path | Role | Description | Criteria |
|---|---|---|---|---|
| `POST` | `/events/{id}/registrations` | Student | Register → CONFIRMED or WAITLISTED | [F3](../criteria/grading-criteria.md#functionality-40-points) · [A1](../criteria/grading-criteria.md#additional-features-25-points) · [S3](../criteria/grading-criteria.md#security-15-points) |
| `DELETE` | `/registrations/{id}` | Student (owner) | Cancel own registration (allowed until event `starts_at`); promotes next waitlisted student | [F3](../criteria/grading-criteria.md#functionality-40-points) · [A1](../criteria/grading-criteria.md#additional-features-25-points) · [S3](../criteria/grading-criteria.md#security-15-points) |
| `GET` | `/registrations/me` | Student | Own registrations + status + waitlist position | [F3](../criteria/grading-criteria.md#functionality-40-points) · [S3](../criteria/grading-criteria.md#security-15-points) |
| `GET` | `/events/{id}/registrations` | Organizer (owner) | Confirmed registrations for own event | [F4](../criteria/grading-criteria.md#functionality-40-points) · [S3](../criteria/grading-criteria.md#security-15-points) |
| `GET` | `/events/{id}/waitlist` | Organizer (owner) | Ordered waitlist (FIFO) for own event | [F4](../criteria/grading-criteria.md#functionality-40-points) · [S3](../criteria/grading-criteria.md#security-15-points) |

---

## Notification jobs

| Method | Path | Role | Description | Criteria |
|---|---|---|---|---|
| `GET` | `/notification-jobs` | Organizer | List jobs for own events (filter by `event_id`) | [A2](../criteria/grading-criteria.md#additional-features-25-points) · [A3](../criteria/grading-criteria.md#additional-features-25-points) |
| `GET` | `/notification-jobs/{id}` | Organizer | Single job detail + delivery status | [A2](../criteria/grading-criteria.md#additional-features-25-points) · [A3](../criteria/grading-criteria.md#additional-features-25-points) |

---

## Profile

| Method | Path | Role | Description |
|---|---|---|---|
| `GET` | `/users/me` | Both | Current user profile |
| `PUT` | `/users/me` | Both | Update display name / email |

---

## Operational

| Method | Path | Role | Description |
|---|---|---|---|
| `GET` | `/health` | Public | Liveness — process is up. Returns `200` unconditionally. |
| `GET` | `/ready` | Public | Readiness — checks Postgres and Redis connectivity. Returns `503` if a dependency is down. |

---

## Error shape

All error responses follow a consistent envelope:

```json
{
    "detail": "Human-readable error message",
    "code": "machine_readable_code"
}
```
