# Phase 8 — Appointment Booking

## 1. Phase Overview

Phase 8 implements the actual appointment booking workflow on top of the dynamic availability and slot generation completed in Phase 7.

The phase introduces appointment creation, appointment reads, faculty acceptance/rejection, availability revalidation, database conflict protection, and transactional notification job intents.

The implementation follows the locked FAS phase plan:

**PLAN → DESIGN → IMPLEMENT → TEST → REVIEW → DOCUMENT → COMMIT/PUSH**

---

## 2. Phase Goal

Enable a student to:

1. View faculty availability.
2. Select a currently generated appointment slot.
3. Provide a reason for the appointment.
4. Submit an appointment request.
5. Have the request created with status `PENDING`.

Faculty members can then view their appointment requests and accept or reject them.

Appointment cancellation, rescheduling, and completion are intentionally deferred to Phase 9.

Full notification delivery infrastructure is intentionally deferred to Phase 10.

---

## 3. Locked Design

### Student booking flow

```text
Student
  ↓
Request faculty availability
  ↓
Receive generated slots
  ↓
Select generated slot
  ↓
Provide reason
  ↓
POST /api/v1/appointments
  ↓
Backend revalidates availability
  ↓
Create PENDING appointment
  ↓
Create notification job intent
```

The frontend is not authoritative for slot validity. The backend revalidates the requested interval against the current generated availability.

There is no `slotId`, because the system does not maintain a persistent slot entity. Slots are generated dynamically from faculty schedules, exceptions, and existing appointments.

---

## 4. API Endpoints

### Create appointment

```http
POST /api/v1/appointments
```

Student-only operation.

The request contains the faculty and the generated appointment interval plus the appointment reason.

### List appointments

```http
GET /api/v1/appointments
```

Visibility is role-based:

- Student → own appointments
- Faculty → appointments belonging to that faculty member
- Admin → all appointments

### Get appointment

```http
GET /api/v1/appointments/:id
```

The backend enforces ownership/role authorization.

### Accept appointment

```http
POST /api/v1/appointments/:id/accept
```

Faculty-only operation.

Valid transition:

```text
PENDING → CONFIRMED
```

### Reject appointment

```http
POST /api/v1/appointments/:id/reject
```

Faculty-only operation.

Valid transition:

```text
PENDING → REJECTED
```

---

## 5. Appointment Status Model

The appointment entity supports:

```text
PENDING
CONFIRMED
REJECTED
CANCELLED
COMPLETED
```

Phase 8 actively uses:

```text
PENDING
CONFIRMED
REJECTED
```

The remaining lifecycle states are reserved for Phase 9.

### Valid Phase 8 transitions

```text
PENDING → CONFIRMED
PENDING → REJECTED
```

Other transitions are rejected during this phase.

---

## 6. Availability Revalidation

When an appointment is created, the service:

1. Verifies the authenticated user is a student.
2. Resolves the student's profile.
3. Verifies the faculty exists.
4. Parses the requested interval.
5. Generates the faculty's current available slots for the requested date.
6. Requires the requested start/end interval to exactly match one generated slot.
7. Rejects arbitrary durations or stale/unavailable slots.
8. Creates the appointment transactionally.

This ensures the frontend cannot bypass availability rules.

### Existing appointments and generated availability

Phase 7 availability generation was extended to consider active appointments.

The following statuses block a generated slot:

```text
PENDING
CONFIRMED
```

Rejected/cancelled/completed appointments do not block future availability.

---

## 7. Availability Exceptions

Availability exceptions continue to be enforced during slot generation.

Therefore, a booking request cannot successfully use a slot that is currently removed by a faculty availability exception.

This preserves the Phase 7 availability model.

---

## 8. Authorization

JWT authentication supplies the current user identity.

### Student

Students may:

- create appointments for themselves
- view their own appointments
- retrieve their own appointment details

A student cannot view another student's appointment.

### Faculty

Faculty members may:

- view appointments belonging to themselves
- accept their own pending appointment requests
- reject their own pending appointment requests

A faculty member cannot manage another faculty member's appointments.

### Admin

Administrators may access appointment records according to the established RBAC rules.

---

## 9. Concurrency and Database Protection

Appointment creation is performed inside a database transaction.

The existing PostgreSQL exclusion constraint remains the final database-level authority for overlapping appointment intervals.

If a concurrent booking reaches the database after another booking has already claimed the interval, PostgreSQL raises exclusion violation code:

```text
23P01
```

The service maps this database conflict to an API-level conflict response rather than exposing the raw database error.

This provides defense in depth:

```text
Backend availability validation
          +
Database transaction
          +
PostgreSQL exclusion constraint
```

---

## 10. Transactional Notification Job Intents

Phase 8 does not implement the full notification worker/email system.

Instead, appointment events create durable notification job intents within the same transaction as the appointment operation.

Implemented notification types include:

```text
APPOINTMENT_REQUESTED
APPOINTMENT_CONFIRMED
APPOINTMENT_REJECTED
```

The notification job records:

- notification type
- recipient
- appointment ID payload
- `PENDING` status
- attempt count
- availability timestamp

The full processing and delivery infrastructure is deferred to Phase 10.

---

## 11. Appointment Acceptance and Rejection

Faculty acceptance/rejection uses a transaction and pessimistic write locking on the appointment record.

The service verifies:

1. The appointment exists.
2. The authenticated faculty member owns the appointment.
3. The appointment is currently `PENDING`.
4. The requested transition is valid.
5. The appointment status is updated.
6. The corresponding notification job intent is created.

Invalid transitions are rejected.

For example:

```text
CONFIRMED → REJECTED
```

is not allowed.

---

## 12. Implementation Sequence

### P8.1 — Appointment service foundation
Created the appointment service structure and dependencies.

### P8.2 — DTO
Added appointment request DTO validation and input structure.

### P8.3 — Booking
Implemented student appointment creation with availability revalidation and transactional notification job creation.

### P8.4 — Appointment-aware availability
Updated generated availability to exclude slots occupied by active appointments.

### P8.5 — Reads
Implemented role-aware appointment listing and individual appointment retrieval.

### P8.6 — Accept/reject
Implemented faculty acceptance and rejection with authorization, valid-state checking, locking, and notification job intents.

### P8.7 — Tests
Expanded appointment service tests and availability regression coverage.

### P8.8 — Final verification
Verified build, full test suite, project structure, and Phase 8 acceptance criteria.

---

## 13. Testing and Verification

Final backend verification:

```text
npm run build
```

Result:

```text
PASS
```

Full test command:

```text
npm test -- --runInBand
```

Result:

```text
13 test suites passed
147 tests passed
0 snapshots
```

Final result:

**147/147 tests passed.**

Phase 7 availability regression coverage also remains passing.

---

## 14. Acceptance Criteria

| ID | Requirement | Result |
|---|---|---|
| P8-01 | Student submits appointment request | PASS |
| P8-02 | JWT student identity | PASS |
| P8-03 | Appointment starts as `PENDING` | PASS |
| P8-04 | Only generated slots accepted | PASS |
| P8-05 | Availability exceptions respected | PASS |
| P8-06 | Existing active appointments block slots | PASS |
| P8-07 | Arbitrary durations rejected | PASS |
| P8-08 | Database concurrency protection | PASS |
| P8-09 | Database conflict mapped to API error | PASS |
| P8-10 | Student sees own appointments | PASS |
| P8-11 | Student cannot view another student's appointments | PASS |
| P8-12 | Faculty sees own appointment requests | PASS |
| P8-13 | Faculty accepts appointment | PASS |
| P8-14 | Faculty rejects appointment | PASS |
| P8-15 | Invalid appointment transitions rejected | PASS |
| P8-16 | Notification job intent created transactionally | PASS |
| P8-17 | Transactional booking / rollback behavior | QUALIFIED |
| P8-18 | Phase 7 regression | PASS |
| P8-19 | Full tests pass | PASS |
| P8-20 | Backend build passes | PASS |

---

## 15. P8-17 Qualification

The production appointment service uses a real TypeORM database transaction for booking and for faculty accept/reject operations.

However, the current appointment unit tests mock the `DataSource.transaction()` method.

Therefore, the tests verify transaction flow and error propagation, but they do not directly prove an actual PostgreSQL rollback.

The correct conclusion is:

> Transactional behavior is implemented in production code. Unit tests verify transaction error handling, but actual PostgreSQL rollback behavior is not directly integration-tested.

This is a testing-coverage limitation, not a known production implementation defect.

A future integration test against a real PostgreSQL database can provide direct rollback verification.

---

## 16. Final Review

Phase 8 implementation satisfies the intended appointment booking architecture.

Important safeguards are present:

- authenticated role enforcement
- ownership checks
- backend availability revalidation
- exception-aware slot generation
- active appointment conflict detection
- PostgreSQL exclusion constraint
- database conflict mapping
- transactional appointment operations
- pessimistic locking for faculty state transitions
- transactional notification job intents
- comprehensive unit/regression test coverage

No cancellation, rescheduling, completion, or full notification delivery system was introduced because those belong to later phases.

---

## 17. Phase 8 Status

**Implementation:** Complete

**Testing:** Complete — 147/147 tests passing

**Build:** Passing

**Review:** Complete

**Documentation:** Complete

**P8.17:** Qualified due to lack of real-database rollback integration coverage

**Phase status:** Ready for commit/push

---

## 18. Next Phase

After the Phase 8 documentation is committed and pushed, begin:

**Phase 9 — Appointment Lifecycle**

Phase 9 will handle the appointment states and operations intentionally deferred from Phase 8, including cancellation, rescheduling, and completion according to the locked project roadmap.
