# FAS - Faculty Appointment System

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

## Development Plan

1. Project planning & requirements
2. Architecture
3. Database design
4. Backend foundation
5. Authentication & authorization
6. Faculty & department management
7. Availability & slot system
8. Appointment booking
9. Appointment lifecycle
10. Notifications & background jobs
11. Frontend foundation
12. Student UI
13. Faculty UI
14. Admin UI
15. UI/UX refinement & design system polish
16. Validation, security & error handling
17. Testing
18. Dockerization
19. Deployment
20. CI/CD
21. Final production hardening

## Documentation

- [Requirements](requirements.md)
- [Architecture](architecture.md)
- [Database Design](database-design.md)
- [Phase 4 - Backend Foundation](phase-4-backend-foundation.md)
- [Phase 5 - Authentication & Authorization](phase-5-authentication-authorization.md)
- [Phase 6 - Faculty & Department Management](phase-6-faculty-department-management.md)
- [Phase 7 - Availability & Slot System](phase-7-availability-slot-system.md)
- [Phase 8 - Appointment Booking](phase-8-appointment-booking.md)
- [Phase 9 - Appointment Lifecycle](phase-9-appointment-lifecycle.md)
- [Phase 10 - Notifications & Background Jobs](phase-10-notifications-background-jobs.md)
- [Phase 11 - Frontend Foundation](phase-11-frontend-foundation.md)
- [Phase 12 - Student UI](phase-12-student-ui.md)
- [Phase 13 - Faculty UI](phase-13-faculty-ui.md)
- [Phase 14 - Admin UI](phase-14-admin-ui.md)

## Project Principles

- Backend enforces all business rules and authorization.
- PostgreSQL is the source of truth.
- Critical appointment operations use database transactions.
- Appointment conflicts are protected at the database level.
- Notifications are processed asynchronously.
- APIs are versioned under `/api/v1`.

## Project Status

**Phase 14 - Admin UI: Completed**

The project is currently progressing through the 21-phase development plan.

Development follows:

**PLAN → DESIGN → IMPLEMENT → TEST → REVIEW → DOCUMENT → COMMIT/PUSH → NEXT PHASE**
