# Phase 9 — Appointment Lifecycle

## Status

**Phase:** 9 — Appointment Lifecycle  
**Workflow:** PLAN → DESIGN → IMPLEMENT → TEST → REVIEW → DOCUMENT → COMMIT/PUSH  
**Status:** Implementation and testing complete; documentation ready for commit.

## Objective

Extend the existing appointment domain with the remaining lifecycle operations:

- Faculty cancellation
- Faculty rescheduling
- Faculty completion

Phase 8 appointment creation and faculty accept/reject behavior remains unchanged.

## Scope

### Included

- Cancel a confirmed appointment
- Reschedule a confirmed appointment to another generated available slot
- Complete a confirmed appointment
- Preserve the existing appointment identity during rescheduling
- Create durable notification job intents for lifecycle events
- Enforce faculty ownership and lifecycle state rules
- Preserve Phase 8 behavior and regression coverage

### Not Included

- Notification delivery workers
- Email delivery
- Student cancellation
- Student rescheduling
- Detailed appointment audit-history tables
- `NO_SHOW` status

Notification delivery remains deferred to Phase 10.

## Appointment Status Model

The appointment statuses are:

- `PENDING`
- `CONFIRMED`
- `REJECTED`
- `CANCELLED`
- `COMPLETED`

Supported lifecycle transitions:

```text
PENDING   → CONFIRMED
PENDING   → REJECTED

CONFIRMED → CANCELLED
CONFIRMED → COMPLETED

CONFIRMED → CONFIRMED
            (reschedule; time changes)
```

Rejected and cancelled appointments remain stored and do not block future availability.

## Authorization Rules

Lifecycle mutation is faculty-only.

- Students may not cancel, reschedule, or complete appointments.
- A faculty member may mutate only appointments belonging to that faculty profile.
- Administrators retain the existing administrative visibility/management capabilities but Phase 9 lifecycle mutation endpoints are implemented as faculty lifecycle operations.
- Backend authorization is enforced by the service layer rather than relying only on the client.

## Cancellation

### Endpoint

```http
POST /api/v1/appointments/:id/cancel
```

### Behavior

1. Require a faculty user.
2. Resolve the faculty profile.
3. Start a database transaction.
4. Lock the appointment row for update.
5. Verify that the appointment exists.
6. Verify that it belongs to the requesting faculty member.
7. Require status `CONFIRMED`.
8. Change the status to `CANCELLED`.
9. Create an `APPOINTMENT_CANCELLED` notification job for the student.
10. Commit the transaction.

The appointment is never deleted.

## Rescheduling

### Endpoint

```http
POST /api/v1/appointments/:id/reschedule
```

### Request

```json
{
  "startTime": "2026-09-07T10:00:00.000Z",
  "endTime": "2026-09-07T10:30:00.000Z"
}
```

Both values must be offset-aware datetimes and the start must be earlier than the end.

### Behavior

1. Require a faculty user.
2. Resolve the faculty profile.
3. Start a database transaction.
4. Lock the appointment row for update.
5. Verify that the appointment exists.
6. Verify faculty ownership.
7. Require status `CONFIRMED`.
8. Generate current available slots for the requested date.
9. Exclude the appointment currently being rescheduled from the blocking-appointment query.
10. Require the requested start/end pair to match a generated available slot.
11. Update the existing appointment's `startTime` and `endTime`.
12. Preserve the appointment ID, student, faculty, reason, and appointment record.
13. Create an `APPOINTMENT_RESCHEDULED` notification job for the student.
14. Commit the transaction.

If the database reports PostgreSQL exclusion-constraint error `23P01`, the service maps it to an appointment conflict.

### Self-exclusion

Rescheduling must not make an appointment conflict with itself.

`getAvailableSlots()` therefore supports an optional:

```ts
excludeAppointmentId?: string
```

When supplied, the appointment query excludes that appointment ID while continuing to treat `PENDING` and `CONFIRMED` appointments as blocking.

## Completion

### Endpoint

```http
POST /api/v1/appointments/:id/complete
```

### Behavior

1. Require a faculty user.
2. Resolve the faculty profile.
3. Start a database transaction.
4. Lock the appointment row for update.
5. Verify that the appointment exists.
6. Verify faculty ownership.
7. Require status `CONFIRMED`.
8. Change the status to `COMPLETED`.
9. Create an `APPOINTMENT_COMPLETED` notification job for the student.
10. Commit the transaction.

## Controller Design

The Phase 9 controller endpoints remain thin:

```text
POST /api/v1/appointments/:id/cancel
POST /api/v1/appointments/:id/reschedule
POST /api/v1/appointments/:id/complete
```

Business rules remain in `AppointmentsService`.

The reschedule endpoint validates the request with the reschedule DTO schema before calling the service.

## Availability Integration

Generated slots remain the authoritative source for valid appointment times.

Rescheduling:

- respects recurring availability
- respects availability exceptions
- respects existing blocking appointments
- uses generated slots rather than storing persistent slot records
- excludes the appointment being rescheduled from its own blocking query

The existing Phase 8 blocking rule remains:

```text
PENDING + CONFIRMED = blocking
REJECTED + CANCELLED = non-blocking
```

## Notification Job Intents

Phase 9 adds durable notification job intents for:

```text
APPOINTMENT_CANCELLED
APPOINTMENT_RESCHEDULED
APPOINTMENT_COMPLETED
```

These jobs are created transactionally with the appointment lifecycle operation.

Actual notification delivery is intentionally deferred to Phase 10.

## Transaction and Concurrency Protection

Critical lifecycle operations use database transactions and pessimistic write locking on the appointment row.

This protects state transitions from concurrent lifecycle mutations.

Rescheduling also retains the PostgreSQL exclusion constraint as the final database-level protection against overlapping appointments.

## API Summary

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/v1/appointments/:id/cancel` | Cancel a confirmed appointment |
| POST | `/api/v1/appointments/:id/reschedule` | Move a confirmed appointment to another generated slot |
| POST | `/api/v1/appointments/:id/complete` | Mark a confirmed appointment completed |

## Testing

Phase 9 regression and feature testing covers:

### Cancellation

- successful cancellation
- student forbidden
- missing faculty profile
- missing appointment
- wrong faculty ownership
- invalid appointment status
- student profile lookup failure
- notification-job failure

### Rescheduling

- successful rescheduling
- student forbidden
- missing faculty profile
- missing appointment
- wrong faculty ownership
- invalid appointment status
- requested time not being a generated slot
- current appointment self-exclusion
- PostgreSQL `23P01` conflict
- student profile lookup failure
- notification-job failure

### Completion

- successful completion
- student forbidden
- missing faculty profile
- missing appointment
- wrong faculty ownership
- invalid appointment status
- student profile lookup failure
- notification-job failure

### Availability regression

The availability service also verifies that the current appointment can be excluded during rescheduling while normal slot generation and appointment blocking behavior remain intact.

## Verification Result

Full Jest test suite:

```text
Test Suites: 13 passed, 13 total
Tests:       175 passed, 175 total
Snapshots:   0 total
```

Phase 9 therefore completed implementation and regression testing without breaking the existing Phase 1–8 backend behavior covered by the test suite.

## Files Changed in Phase 9

Primary backend implementation:

```text
src/appointments/dto/appointment.dto.ts
src/appointments/appointments.service.ts
src/appointments/appointments.controller.ts
src/availability-schedules/availability-schedules.service.ts
```

Tests:

```text
src/appointments/appointments.service.spec.ts
src/availability-schedules/availability-schedules.service.spec.ts
```

## Review Outcome

Phase 9 satisfies the planned lifecycle scope:

- Cancellation is implemented as `CONFIRMED → CANCELLED`.
- Rescheduling updates the existing appointment rather than creating a new appointment.
- Completion is implemented as `CONFIRMED → COMPLETED`.
- Faculty ownership and lifecycle state are enforced.
- Rescheduling validates against generated availability and excludes the appointment itself.
- Notification intents are durable and transactional.
- Phase 8 appointment creation and accept/reject behavior remains covered by regression tests.
- Full automated tests pass: **175/175**.

## Phase Completion

**Phase 9 implementation:** Complete  
**Phase 9 testing:** Complete  
**Phase 9 review:** Complete  
**Phase 9 documentation:** Complete  

Next required workflow step:

```text
COMMIT / PUSH
```

After the Phase 9 documentation and implementation are committed and pushed, begin Phase 10 in a new chat:

**06Plan Phase 10 Notifications & Background Jobs**
