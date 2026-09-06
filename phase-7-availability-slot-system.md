# FAS — Phase 7: Availability & Slot System

## Phase Objective

Phase 7 implements the faculty availability and slot-generation capability required by the Faculty Appointment System.

The system allows faculty members and administrators to manage recurring faculty appointment availability and temporary unavailability exceptions.

Available appointment slots are generated dynamically from these rules rather than being stored as individual slot records.

Appointment booking and appointment lifecycle management are intentionally deferred to later phases.

---

## Scope

### Included

- Faculty availability schedules
- Faculty availability exceptions
- Dynamic slot generation
- Student-facing availability reads
- Faculty availability management
- Authorization for availability management
- Validation of availability inputs
- Availability-related automated tests

### Excluded

The following are not part of Phase 7:

- Appointment booking
- Appointment confirmation or rejection
- Appointment cancellation
- Appointment rescheduling
- Appointment completion
- Notifications
- Background notification jobs
- Institute calendar or holiday management
- Stored individual appointment-slot records

---

# Step 7.1: Faculty Availability Schedules

## What we're building

Faculty availability is represented using recurring weekly availability schedules.

A schedule defines a period during which a faculty member normally accepts appointments.

Each schedule contains:

- Faculty
- Day of week
- Start time
- End time
- Slot duration
- Active/inactive state

The database entity is:

`availability_schedules`

The schedule is a recurring availability rule rather than a faculty teaching timetable.

For example:

```text
Monday
09:00–12:00
30-minute slots
```

produces the logical appointment slots:

```text
09:00–09:30
09:30–10:00
10:00–10:30
10:30–11:00
11:00–11:30
11:30–12:00
```

Slots are generated when availability is requested rather than stored permanently.

---

## Schedule Validation

Availability schedules enforce the following rules:

- Day of week must be between 0 and 6.
- Start time must be earlier than end time.
- Slot duration must be one of:
  - 15 minutes
  - 30 minutes
  - 45 minutes
  - 60 minutes
- Inactive schedules are ignored when generating availability.
- Overlapping schedules for the same faculty and day are rejected.
- Adjacent schedules are allowed.

For example:

```text
09:00–10:00
10:00–11:00
```

is valid because the schedules are adjacent rather than overlapping.

Database-level checks already enforce the schedule time range and allowed slot durations.

---

# Step 7.2: Availability Exceptions

## What we're building

Faculty availability exceptions represent temporary periods when a faculty member is unavailable despite having a normal recurring availability schedule.

The database entity is:

`availability_exceptions`

Exceptions therefore represent **unavailability**, not additional availability.

An exception may be:

### Whole-day

```text
date = 2026-09-07
start_time = NULL
end_time = NULL
```

This removes the faculty member's availability for the entire date.

### Partial-day

```text
date = 2026-09-07
start_time = 10:00
end_time = 11:00
```

This removes only the specified period from the normal availability.

---

## Exception Rules

The following rules apply:

- Whole-day exceptions use NULL for both start and end time.
- Partial-day exceptions must provide both start and end time.
- Partial-day exception start time must be earlier than end time.
- Exceptions outside the faculty member's normal schedule have no effect.
- Multiple overlapping exceptions are handled as a combined unavailable interval.
- An exception may overlap the beginning or end of a schedule.

No institute-wide holiday or calendar system is introduced in this phase.

Faculty-specific exceptions are sufficient for the current requirements.

---

# Step 7.3: Dynamic Slot Generation

## What we're building

The system generates appointment slots dynamically from recurring availability schedules.

Slots are not stored in a separate database table.

The calculation is:

```text
Recurring Schedule
        ↓
Applicable Date
        ↓
Availability Exceptions
        ↓
Effective Available Intervals
        ↓
Generate Fixed Slot Grid
        ↓
Return Available Slots
```

---

## Fixed Slot Grid

Slots remain aligned to the original schedule.

For example:

```text
Schedule:
09:00–12:00

Duration:
30 minutes
```

The slot grid is always:

```text
09:00–09:30
09:30–10:00
10:00–10:30
10:30–11:00
11:00–11:30
11:30–12:00
```

Exceptions remove slots from this fixed grid.

They do not create a new slot-generation starting point.

---

## Non-Aligned Exceptions

This behavior was explicitly verified during Phase 7.

Example:

```text
Schedule:
09:00–12:00

Slot duration:
30 minutes

Exception:
10:15–10:45
```

The original slot grid remains:

```text
09:00–09:30
09:30–10:00
10:00–10:30
10:30–11:00
11:00–11:30
11:30–12:00
```

The slots intersecting the unavailable interval are removed.

The result is:

```text
09:00–09:30
09:30–10:00
11:00–11:30
11:30–12:00
```

The system does not shift the grid to:

```text
10:45–11:15
11:15–11:45
```

This preserves predictable appointment slot boundaries.

---

## Complete-Slot Rule

A slot is returned only when the entire slot is contained within an effective available interval.

Therefore, if an exception cuts through a slot, that slot is removed rather than shortened.

For example:

```text
Slot:
10:00–10:30

Exception:
10:15–10:45
```

The slot is removed because the complete slot is not available.

---

## Leftover Time

If a schedule is not evenly divisible by its slot duration, leftover time is discarded.

Example:

```text
Schedule:
09:00–10:20

Duration:
30 minutes
```

Generated slots:

```text
09:00–09:30
09:30–10:00
10:00–10:30
```

The final slot would exceed the schedule boundary, so the implementation stops before generating it.

---

# Step 7.4: Availability API

The availability module exposes operations for managing schedules and reading generated availability.

The API uses the existing `/api/v1` versioning convention.

### Schedule Management

The availability schedule API supports:

- Creating a faculty availability schedule
- Listing schedules for a faculty member
- Retrieving an individual schedule
- Updating a schedule
- Removing a schedule

### Availability Read

The system provides a faculty availability endpoint that accepts:

```text
facultyId
date
```

and returns the dynamically generated available slots for that date.

The availability read is based on:

```text
active recurring schedules
+
matching availability exceptions
```

At this stage the result represents **schedule availability**.

It does not yet perform appointment-booking conflict checks.

Actual booking behavior belongs to Phase 8.

---

# Step 7.5: Authorization

Availability management follows the existing authentication and RBAC model.

### Faculty

A faculty member may manage their own availability.

A faculty member may not manage another faculty member's availability.

### Administrator

Administrators may manage faculty availability.

### Student

Students may read availability but may not create, update, or remove faculty availability.

Authorization is enforced in the service layer rather than relying only on the frontend.

---

# Step 7.6: Validation and Database Integrity

Input validation is performed using Zod schemas.

Validation includes:

- UUID validation for faculty identifiers
- Day-of-week validation
- Time format validation
- Start/end ordering
- Allowed slot durations
- Required update-field behavior
- Date format validation

Availability exception validation follows the whole-day/partial-day rules.

The database also enforces important invariants including:

```text
availability_schedules.start_time < availability_schedules.end_time
```

and:

```text
slot_duration IN (15, 30, 45, 60)
```

Availability exception time ranges are also protected by database constraints.

---

# Step 7.7: Testing

Phase 7 was validated using automated unit tests and a complete backend build.

## Availability Schedule Service

The schedule service test suite covers:

- Normal slot generation
- Non-aligned exception boundaries
- Partial-day exceptions
- Whole-day exceptions
- Multiple exceptions
- Overlapping exceptions
- Exceptions outside schedules
- Exceptions overlapping schedule boundaries
- Leftover schedule time
- Inactive schedules
- Missing schedules
- Schedule retrieval
- Schedule creation
- Schedule overlap detection
- Adjacent schedules
- RBAC
- Schedule updates
- Schedule removal

Result:

```text
26 tests passed
```

## Full Backend Test Suite

Final backend verification:

```text
Test Suites: 12 passed, 12 total
Tests:       107 passed, 107 total
```

## Build

The NestJS backend build completed successfully:

```text
npm run build
```

Result:

```text
Build: passed
```

---

# Important Design Decisions

## 1. Recurring availability instead of manually stored slots

Faculty define normal availability windows and a slot duration.

The system generates individual slots dynamically.

This avoids storing a large number of redundant slot records.

---

## 2. Exceptions represent unavailability

The normal schedule represents when appointments are available.

Exceptions represent temporary periods when appointments are unavailable.

This keeps the domain model aligned with the existing database design.

---

## 3. Fixed slot alignment

Generated slots are anchored to the original schedule boundaries.

Exceptions filter the fixed slot grid instead of changing its alignment.

This prevents unpredictable slot boundaries.

---

## 4. No calendar/holiday subsystem

Institute-wide calendar and holiday functionality is outside the current scope.

Faculty-specific availability exceptions provide the required mechanism for temporary changes to availability.

---

## 5. No appointment logic in Phase 7

Availability is implemented independently from appointment booking.

Appointment conflict detection and booking behavior will be introduced in Phase 8.

---

# Files Added or Modified

The Phase 7 backend implementation includes:

```text
backend/src/availability-schedules/
backend/src/availability-exceptions/
```

The modules contain the relevant:

- Entities
- DTOs
- Controllers
- Services
- Unit tests

The availability modules were also integrated into the backend module configuration.

---

# Phase 7 Review Result

Phase 7 functionality was reviewed against the agreed design.

The review confirmed:

- Recurring availability is correctly modeled.
- Exceptions correctly represent temporary unavailability.
- Slot generation is dynamic.
- Slot boundaries remain aligned to the original schedule.
- Partial and whole-day exceptions are handled.
- Overlapping exceptions are handled logically.
- Authorization rules are enforced.
- Availability validation is implemented.
- Appointment booking remains outside the phase scope.
- Calendar/holiday functionality remains outside the phase scope.

A non-aligned exception case was identified during review and corrected before final verification.

The correction was verified by the dedicated regression test.

---

# Phase 7 Completion

Phase 7 — Availability & Slot System is complete.

Final verification:

```text
Availability schedule tests: 26/26 passed
Full backend tests:          107/107 passed
Backend build:               passed
```

The availability system now provides the backend foundation required for appointment booking.

The next implementation phase is:

**Phase 8 — Appointment Booking**
