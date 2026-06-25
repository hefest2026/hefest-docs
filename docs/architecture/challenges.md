# Challenges & Lessons Learned

> **Covers:** [F3](../criteria/grading-criteria.md#functionality-40-points) (concurrency correctness) · [F4](../criteria/grading-criteria.md#functionality-40-points) (deterministic FIFO) · [SC2](../criteria/grading-criteria.md#scalability-design-15-points) (query optimization) · [CQ3](../criteria/grading-criteria.md#code-quality-10-points)

The happy path of registrations is easy. The hard part is everything that happens **when two students click at the same millisecond.** These are the real problems we hit on the registration + waitlist core and how we resolved them.

---

## 1. The double-cancel race

**Symptom.** A student double-clicks "Cancel". Two requests arrive almost together.

Our first version read the registration, *then* acquired the `FOR UPDATE` lock on the event:

1. Both requests read the registration and see `status = confirmed`.
2. Request A takes the event lock, cancels the registration, promotes the next waitlisted student, commits, releases the lock.
3. Request B now takes the lock — but acts on its **stale in-memory copy** that still says `confirmed`. It cancels an already-cancelled row and promotes a **second** waitlisted student who never should have moved.

**Fix.** After acquiring the event lock, re-read the registration with `SELECT FOR UPDATE` and re-check its status against committed state before deciding anything:

```python
event = await Event.filter(id=reg.event_id).using_db(conn).select_for_update().get_or_none()
# Re-read under the lock — committed state, not the pre-lock snapshot.
reg = await Registration.filter(id=reg_id).using_db(conn).select_for_update().get()
if reg.status == RegistrationStatus.cancelled:
    raise HTTPException(409, "registration already cancelled")
```

**Lesson.** *A pessimistic lock only protects you if you re-read the data after you hold it.* Reading before the lock and acting on that snapshot afterwards defeats the entire point of locking.

---

## 2. Identical timestamps broke FIFO

**Symptom.** Waitlist promotion ordered purely by `registered_at`. Under load, two registrations can land on the exact same microsecond, and PostgreSQL gives **no ordering guarantee** when the sort key ties. Two problems followed:

- "Next in line" became non-deterministic — a different student could be promoted on each run.
- The position number shown to the student (`/registrations/me`) was computed a *different* way than the promotion order, so the two could disagree: a student shown "position 1" might not be the one promoted.

**Fix.** Add `id` as a deterministic tie-breaker **everywhere** the waitlist is ordered — promotion, the organizer `/waitlist` view, and the position calculation:

```python
.order_by("registered_at", "id")
```

**Lesson.** *Timestamps are not unique keys.* Any "first in line" ordering needs a stable tie-breaker, and every place that reads the order must use the same one, or the UI and the behavior drift apart.

---

## 3. The "phantom" we deliberately did *not* fix

**Context.** A code review flagged two "critical" issues: overbooking via a phantom read, and a missing unique constraint to enforce one registration per event.

**What we verified.** Neither held up against the actual code:

- **Overbooking** — every writer (`register_student`, and the promotion inside `cancel_registration`) takes `SELECT FOR UPDATE` on the **event row first**. That row is the serialization gate: a second transaction blocks there until the first commits, so the seat count is never stale. No advisory lock or `SERIALIZABLE` isolation is needed.
- **Unique constraint** — it *did* exist, in migration `0002`, as a **partial** constraint: `UNIQUE (event_id, student_id) WHERE status IN ('confirmed','waitlisted')`. The reviewer had only read `0001_initial.py`. The partial form also solves a follow-on trap for free: because cancelled rows are excluded, a student who cancels can **re-register** without hitting a duplicate-key error.

**Lesson.** *External review is a set of hypotheses to verify, not orders to follow.* We evaluated each claim against the schema and the locking model and pushed back with evidence rather than implementing changes that would have added complexity without fixing a real bug.

---

## 4. An N+1 in `/registrations/me`

**Symptom.** Computing each waitlist position ran one `COUNT(*)` query per waitlisted registration. A student waitlisted for 20 events triggered 21 queries.

**Fix.** Collapse it into a **single grouped query** that fetches all waitlisted rows for the relevant events in FIFO order, then assign positions in one pass in application code. The query rides the existing `idx_registrations_waitlist_fifo` partial index.

**Lesson.** *A query inside a loop is a query you can usually hoist out of it.* One ordered read plus an in-memory counter replaces N round-trips.

---

## The one-line takeaway for the demo

> The challenge wasn't registering a student for an event — it was guaranteeing that a capacity-3 event never has a 4th confirmed seat, and that the waitlist promotes exactly one correct person, even when requests collide.
