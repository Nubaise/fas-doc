# FAS — Database Design

## 1. Database

**PostgreSQL**

TypeORM is used as the ORM and database migration tool.

The database is the source of truth for application data and appointment integrity.

## 2. Core Entities

```text
users
departments
students
faculty
availability_schedules
availability_exceptions
appointments
notification_jobs
notifications
```

## 3. Users

```text
users
-----
id              UUID PK
email           VARCHAR UNIQUE
password_hash   VARCHAR
role            ENUM(STUDENT, FACULTY, ADMIN)
is_active       BOOLEAN
created_at      TIMESTAMP
updated_at      TIMESTAMP
```

## 4. Departments

```text
departments
-----------
id          UUID PK
name        VARCHAR UNIQUE
code        VARCHAR UNIQUE
created_at  TIMESTAMP
updated_at  TIMESTAMP
```

## 5. Students

```text
students
--------
id               UUID PK
user_id          UUID FK UNIQUE
student_number   VARCHAR UNIQUE
first_name       VARCHAR
last_name        VARCHAR
department_id    UUID FK
created_at       TIMESTAMP
updated_at       TIMESTAMP
```

Relationships:

```text
users ──────── students
departments ─ students
```

## 6. Faculty

```text
faculty
-------
id               UUID PK
user_id          UUID FK UNIQUE
employee_number  VARCHAR UNIQUE
first_name       VARCHAR
last_name        VARCHAR
department_id    UUID FK
created_at       TIMESTAMP
updated_at       TIMESTAMP
```

Relationships:

```text
users ──────── faculty
departments ─ faculty
```

## 7. Availability Schedules

```text
availability_schedules
----------------------
id               UUID PK
faculty_id       UUID FK
day_of_week      SMALLINT
start_time       TIME
end_time         TIME
slot_duration    INTEGER
is_active        BOOLEAN
created_at       TIMESTAMP
updated_at       TIMESTAMP
```

`day_of_week` represents the recurring day.

`slot_duration` is the duration in minutes and supports:

```text
15
30
45
60
```

Database integrity rules:

- `start_time` must be earlier than `end_time`.
- `slot_duration` must be one of `15`, `30`, `45`, or `60`.

Example:

```text
Monday
09:00 → 12:00
30 minute duration
```

generates available slots such as:

```text
09:00 → 09:30
09:30 → 10:00
10:00 → 10:30
...
```

Slots are generated logically from availability and are not permanently stored in a separate slot table.

## 8. Availability Exceptions

```text
availability_exceptions
-----------------------
id          UUID PK
faculty_id  UUID FK
date        DATE
start_time  TIME NULL
end_time    TIME NULL
reason      VARCHAR NULL
created_at  TIMESTAMP
```

Exceptions temporarily modify recurring availability.

Examples:

**Whole-day exception**

```text
date       = 2026-09-15
start_time = NULL
end_time   = NULL
```

**Partial-day exception**

```text
date       = 2026-09-15
start_time = 10:00
end_time   = 12:00
```

Database integrity rules:

- Whole-day exceptions require both `start_time` and `end_time` to be `NULL`.
- Partial-day exceptions require both times to be provided.
- For partial-day exceptions, `start_time` must be earlier than `end_time`.

## 9. Appointments

```text
appointments
------------
id           UUID PK
student_id   UUID FK
faculty_id   UUID FK
start_time   TIMESTAMP
end_time     TIMESTAMP
reason       TEXT
status       ENUM
created_at   TIMESTAMP
updated_at   TIMESTAMP
```

Appointment statuses:

```text
PENDING
CONFIRMED
REJECTED
CANCELLED
COMPLETED
```

Database integrity rules:

- `start_time` must be earlier than `end_time`.
- `PENDING` and `CONFIRMED` appointments cannot overlap for the same faculty member.

Relationships:

```text
students ───── appointments
faculty  ───── appointments
```

## 10. Appointment Integrity

A faculty member must not have overlapping active appointments.

Active appointment statuses are:

```text
PENDING
CONFIRMED
```

The database must enforce this rule so that concurrent requests cannot create conflicting appointments.

The implementation uses a PostgreSQL GiST exclusion constraint with a half-open time range `[start_time, end_time)`.

Appointment creation and state changes must use database transactions where required.

## 11. Notification Jobs

```text
notification_jobs
-----------------
id             UUID PK
type           VARCHAR
recipient_id   UUID FK
payload        JSONB
status         ENUM(PENDING, PROCESSING, COMPLETED, FAILED)
attempts       INTEGER
available_at   TIMESTAMP
processed_at   TIMESTAMP NULL
created_at     TIMESTAMP
updated_at     TIMESTAMP
```

Notification jobs provide durable asynchronous processing.

The worker processes pending jobs and sends the corresponding notification.

## 12. Notifications

```text
notifications
-------------
id          UUID PK
user_id     UUID FK
type        VARCHAR
title       VARCHAR
message     TEXT
read_at     TIMESTAMP NULL
created_at  TIMESTAMP
```

Notifications provide an in-application record of important events.

## 13. Relationships

```text
                    ┌──────────────┐
                    │    users     │
                    └──────┬───────┘
                           │
                ┌──────────┴──────────┐
                │                     │
                ▼                     ▼
          ┌───────────┐         ┌───────────┐
          │ students  │         │  faculty  │
          └─────┬─────┘         └─────┬─────┘
                │                     │
                │                     ├──────────────┐
                │                     │              │
                │                     ▼              ▼
                │             ┌──────────────┐ ┌───────────────┐
                │             │ availability │ │ availability  │
                │             │  schedules   │ │  exceptions   │
                │             └──────────────┘ └───────────────┘
                │
                └──────────────┐
                               │
                               ▼
                       ┌──────────────┐
                       │ appointments │
                       └──────────────┘

          ┌──────────────┐
          │ departments  │
          └──────┬───────┘
                 │
          ┌──────┴──────┐
          ▼             ▼
      students        faculty

users ──────── notifications
users ──────── notification_jobs
```

## 14. Data Integrity

The database should enforce:

- Primary keys
- Foreign keys
- Unique constraints
- Valid status values
- Valid availability periods
- Appointment conflict protection
- Transactional consistency

Schema changes are managed through TypeORM migrations.

Automatic schema synchronization is disabled.

## 15. Design Principles

- No permanent generated slot table.
- Recurring availability defines possible appointment times.
- Appointments store actual booked time ranges.
- Rejected or cancelled appointments do not block future availability.
- Database constraints protect against concurrent booking conflicts.
- Appointment history must be preserved when appointments are rescheduled.
