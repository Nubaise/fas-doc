# FAS — Faculty Appointment System

Documentation repository for the Faculty Appointment System (FAS).

## Purpose

FAS is a web-based system that allows students to request appointments with faculty based on faculty-defined availability.

## Core Features

- Student appointment requests
- Faculty availability management
- Faculty approval and rejection of requests
- Appointment status tracking
- Faculty-controlled cancellation and rescheduling
- Email notifications
- Role-based access control

## User Roles

### Student

- View faculty
- View available appointment slots
- Request appointments
- View appointment status
- Receive notifications

### Faculty

- Manage profile
- Manage availability
- Configure appointment duration
- Manage appointment requests
- Accept or reject requests
- Cancel or reschedule appointments
- Receive notifications

### Admin

- Manage users
- Manage faculty and departments
- Oversee appointments
- Manage system configuration
- View system activity

## Technology Stack

| Layer          | Technology                         |
| -------------- | ---------------------------------- |
| Frontend       | React + TypeScript                 |
| Backend        | NestJS + TypeScript + Node.js      |
| API            | REST + OpenAPI                     |
| Database       | PostgreSQL                         |
| ORM            | TypeORM                            |
| Authentication | JWT                                |
| Authorization  | RBAC                               |
| Notifications  | Transactional email + durable jobs |
| Worker         | Independent background worker      |
| Deployment     | Docker + managed hosting           |
| Infrastructure | Terraform/OpenTofu                 |
| CI/CD          | Managed CI/CD                      |

## Documentation

- [Requirements](requirements.md)
- [Architecture](architecture.md)
- [Database Design](database-design.md)
- [Phase 4 — Backend Foundation](phase-4-backend-foundation.md)
- [Phase 5 — Authentication & Authorization](phase-5-authentication-authorization.md)
- [Phase 6 — Faculty & Department Management](phase-6-faculty-department-management.md)
- [Phase 7 — Availability & Slot System](phase-7-availability-slot-system.md)
- [Phase 8 — Appointment Booking](phase-8-appointment-booking.md)
- [Phase 9 — Appointment Lifecycle](phase-9-appointment-lifecycle.md)
- [Phase 10 — Notifications & Background Jobs](phase-10-notifications-background-jobs.md)
- [Phase 11 — Frontend Foundation](phase-11-frontend-foundation.md)

## Project Principles

- Backend enforces all business rules and authorization.
- PostgreSQL is the source of truth.
- Critical appointment operations use database transactions.
- Appointment conflicts are protected at the database level.
- Notifications are processed asynchronously.
- APIs are versioned under `/api/v1`.

## Project Status

Currently in development.

Development follows:

**PLAN → DESIGN → DOCUMENT → IMPLEMENT → TEST → REVIEW → NEXT PHASE**
