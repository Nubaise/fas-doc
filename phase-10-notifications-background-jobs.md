# FAS — Phase 10: Notifications & Background Jobs

**Status:** Complete  
**Phase:** 10  
**Implementation status:** Implemented, tested, and reviewed  
**Final automated test result:** 183/183 tests passed

---

## 1. Objective

Phase 10 establishes reliable asynchronous notification processing for FAS.

The implementation separates:

- authoritative appointment state changes
- durable notification intent
- asynchronous notification processing
- in-app notification persistence
- email delivery
- retry and failure handling

The database remains the durable source of truth for notification work.

---

## 2. Architecture

The Phase 10 flow is:

```text
Appointment Lifecycle Operation
          |
          | database transaction
          v
    Notification Job
        PENDING
          |
          v
   Independent Worker
          |
          v
      PROCESSING
          |
          +----------------------+
          |                      |
          v                      v
 In-App Notification        Email Delivery
     Persistence            (provider abstraction)
          |                      |
          +----------+-----------+
                     |
              success / failure
                     |
          +----------+----------+
          |                     |
          v                     v
      COMPLETED              retry / FAILED
```

The implementation uses the existing modular-monolith architecture with a separate worker process. A separate notification microservice is not required.

---

## 3. Durable Notification Jobs

The notification job entity is located at:

```text
backend/src/notification-jobs/entities/notification-job.entity.ts
```

The durable `notification_jobs` record contains:

- job ID
- notification type
- recipient
- recipient ID
- JSON payload
- status
- attempt count
- available time
- processed time
- creation timestamp
- update timestamp

The implemented statuses are:

```text
PENDING
PROCESSING
COMPLETED
FAILED
```

The existing status model is preserved rather than introducing an additional `SENT` state.

---

## 4. Transactional Notification Intent

Notification work is created as part of the relevant appointment business transaction.

Conceptually:

```text
Database Transaction
|
+-- Appointment state change
|
+-- Notification Job → PENDING
|
+-- COMMIT
```

This ensures that notification intent is durable only when the associated business operation commits.

Notification delivery itself occurs later and does not determine whether the appointment operation succeeded.

This follows the architectural distinction:

```text
Business Transaction
    = durable, atomic, authoritative

Notification Delivery
    = asynchronous, retryable, eventually delivered
```

---

## 5. Notification Job Processing

The main service is:

```text
backend/src/notification-jobs/notification-jobs.service.ts
```

The worker obtains the next available pending job and processes it.

A job is eligible when:

```text
status = PENDING
available_at <= current time
```

Jobs are ordered by:

1. `available_at`
2. `created_at`

---

## 6. Concurrent Job Claiming

Job claiming is performed inside a PostgreSQL transaction using:

```text
pessimistic_write
SKIP LOCKED
```

The flow is:

```text
PENDING
   |
   v
claim row
   |
   +-- lock row
   |
   +-- skip rows locked by another worker
   |
   +-- change status to PROCESSING
   |
   +-- increment attempts
```

This allows multiple worker processes to operate concurrently without intentionally claiming the same pending row.

---

## 7. Independent Worker

The worker entry point is:

```text
backend/src/worker/main.ts
```

The worker:

1. creates a Nest application context
2. obtains `NotificationJobsService`
3. repeatedly calls `processNext()`
4. waits when no job is available
5. handles `SIGINT` and `SIGTERM`
6. closes the application context during shutdown

The current polling interval is:

```text
1 second
```

The worker is intentionally separate from the HTTP API process.

---

## 8. Notification Content

Notification content is isolated in:

```text
backend/src/notification-content/notification-content.service.ts
```

The implemented appointment notification types are:

```text
APPOINTMENT_REQUESTED
APPOINTMENT_CONFIRMED
APPOINTMENT_REJECTED
APPOINTMENT_CANCELLED
APPOINTMENT_RESCHEDULED
APPOINTMENT_COMPLETED
```

Unsupported notification types are treated as permanent failures.

Notification content is therefore not scattered through the worker implementation.

---

## 9. In-App Notifications

The existing notification entity remains:

```text
backend/src/notifications/entities/notification.entity.ts
```

The persisted notification contains:

- recipient
- notification type
- title
- message
- read timestamp
- creation timestamp

The Phase 10 implementation deliberately keeps the existing notification schema unchanged.

No `notificationJobId` relationship was introduced.

### Database migration decision

No Phase 10 migration is required for the implemented notification-job processing design.

The database migration system remains controlled separately, and migrations must be executed by the project developer rather than implicitly by application startup.

---

## 10. Email Delivery Abstraction

Email delivery is separated behind an interface.

Files:

```text
backend/src/email/email.types.ts
backend/src/email/email-delivery.interface.ts
backend/src/email/mock-email-delivery.service.ts
backend/src/email/email.module.ts
```

The worker depends on the `EmailDelivery` abstraction rather than directly depending on an SMTP or transactional-email provider.

The current implementation uses:

```text
MockEmailDeliveryService
```

This keeps development and automated testing from accidentally sending real production email.

The email message contains an idempotency key:

```text
idempotencyKey = notification job ID
```

This provides groundwork for a production delivery adapter that supports provider-side idempotency.

---

## 11. Retry Policy

Transient processing failures are retried with bounded backoff.

Current implementation:

```text
MAX_ATTEMPTS = 3
BASE_BACKOFF = 5 seconds
```

The resulting delays are:

```text
Attempt 1 failure
    |
    +-- 5 seconds
    v
Attempt 2

Attempt 2 failure
    |
    +-- 10 seconds
    v
Attempt 3

Attempt 3 failure
    |
    v
FAILED
```

The retry schedule is controlled using `availableAt`.

This prevents immediate retry loops and provides bounded recovery behavior.

---

## 12. Permanent Failures

Permanent failures are not retried.

Examples implemented include:

- notification recipient does not exist
- notification recipient is inactive
- notification job is missing an appointment ID
- referenced appointment does not exist
- unsupported notification type

Such jobs are transitioned to:

```text
FAILED
```

---

## 13. Stale Job Recovery

A job left in `PROCESSING` for longer than the configured processing timeout can be recovered.

Current processing timeout:

```text
60 seconds
```

If attempts remain:

```text
PROCESSING
     |
     v
PENDING
```

with a new `availableAt`.

If the job has exhausted its attempts:

```text
PROCESSING
     |
     v
FAILED
```

This prevents permanently abandoned processing rows from remaining stuck indefinitely.

### Known caveat

The current stale-job recovery approach has a small concurrency race around detecting a stale job while another worker may be completing work at approximately the same time.

This is recorded as a future hardening consideration rather than treated as a Phase 10 blocker.

---

## 14. Failure Isolation

Notification delivery failure must not invalidate the appointment business transaction.

The intended separation is:

```text
Appointment transaction
        |
        +-- durable notification intent
        |
        +-- COMMIT
                 |
                 v
          asynchronous worker
                 |
                 +-- email failure
                 |
                 +-- retry
```

Therefore, an email provider outage does not roll back an already committed appointment state transition.

---

## 15. Delivery Semantics

Phase 10 provides **at-least-once processing**.

It does not claim exactly-once external delivery.

The current processing sequence creates the in-app notification before email delivery and marks the job complete only after email delivery succeeds.

Consequently, a transient email failure after notification persistence can cause the job to be retried and another in-app notification to be created.

This is an acknowledged limitation of the current schema and processing design.

Exactly-once email delivery also cannot be guaranteed solely by application code because the external provider boundary is involved.

The email abstraction's `idempotencyKey` provides groundwork for stronger provider-side idempotency where supported.

---

## 16. Application Configuration

The main application enables automatic entity loading:

```text
autoLoadEntities: true
```

This is required so the worker-created Nest application context loads the notification job entity and its related runtime metadata.

The migration data source remains explicitly configured with the project's entity list.

Runtime synchronization remains disabled:

```text
synchronize: false
```

---

## 17. Verification

Phase 10 verification included:

### Automated tests

```text
npm test -- --runInBand
```

Final result:

```text
Test Suites: 14 passed, 14 total
Tests:       183 passed, 183 total
Snapshots:   0 total
```

### Tested behavior

The test suite covers:

- no pending job
- successful notification processing
- successful email delivery
- notification persistence
- separate notifications for separate jobs
- transient email failure
- retry scheduling
- missing recipient
- inactive recipient
- stale job recovery
- exhausted stale job handling

### Worker verification

The independent worker was started during Phase 10 verification and successfully:

- initialized the Nest application context
- loaded `NotificationJobsModule`
- connected to PostgreSQL

The worker was then stopped cleanly.

---

## 18. Review Findings

Phase 10 review result:

**APPROVED**

| Area | Result |
|---|---|
| Durable notification jobs | PASS |
| PostgreSQL-backed job processing | PASS |
| Transactional notification intent | PASS |
| Independent worker | PASS |
| Concurrent job claiming | PASS |
| PostgreSQL `SKIP LOCKED` | PASS |
| Retry/backoff | PASS |
| Maximum attempts | PASS |
| Stale job recovery | PASS with known caveat |
| In-app notifications | PASS |
| Email abstraction | PASS |
| Development-safe email delivery | PASS |
| Six appointment notification types | PASS |
| Failure isolation | PASS |
| Migration requirement | None |
| Automated tests | **183/183 PASS** |
| Exactly-once delivery | Not claimed |

---

## 19. Important Files

```text
backend/src/app.module.ts

backend/src/worker/main.ts

backend/src/notification-jobs/
├── entities/
│   └── notification-job.entity.ts
├── notification-jobs.module.ts
├── notification-jobs.service.ts
└── notification-jobs.service.spec.ts

backend/src/notification-content/
├── notification-content.module.ts
└── notification-content.service.ts

backend/src/email/
├── email.types.ts
├── email-delivery.interface.ts
├── mock-email-delivery.service.ts
└── email.module.ts

backend/src/notifications/
└── entities/
    └── notification.entity.ts
```

---

## 20. Lessons Learned

### Durable intent is different from delivery

The application should first establish durable notification intent as part of the business transaction. Actual delivery belongs to asynchronous processing.

### PostgreSQL can act as the durable job store

For the current FAS architecture, PostgreSQL provides the durable notification-job store without requiring a separate queue product.

### `SKIP LOCKED` supports concurrent workers

PostgreSQL row locking with `SKIP LOCKED` provides a practical mechanism for multiple workers to claim independent jobs.

### Retries must be bounded

Retrying every failure forever can create poison-job loops. The implementation therefore distinguishes permanent failures and limits transient retries.

### Exactly-once should not be promised casually

A worker can provide durable, retryable processing, but external side effects such as email require idempotency support where possible and still cannot be treated as universally exactly-once.

### Existing schemas should not be changed without a real requirement

The original `notifications` schema was sufficient for the Phase 10 implementation, so no unnecessary `notificationJobId` column or migration was introduced.

---

## 21. Phase Completion

Phase 10 has completed the required workflow:

```text
PLAN       ✓
DESIGN     ✓
IMPLEMENT  ✓
TEST       ✓ 183/183
REVIEW     ✓ APPROVED
DOCUMENT   ✓
COMMIT     Pending
PUSH       Pending
```

The next step is to review the generated documentation against the actual repository changes, then commit and push the Phase 10 implementation and documentation.

---

## 22. Next Phase

After Phase 10 is committed and pushed, continue according to the governing FAS implementation plan.

The next phase should be determined from the current implementation plan rather than from the historical/legacy FAS attempt.
