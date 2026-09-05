# FAS — Architecture

## 1. Technology Stack

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
| Logging        | Structured logging                 |

## 2. High-Level Architecture

```text
┌──────────────┐
│    Users     │
└──────┬───────┘
       │
       ▼
┌────────────────────┐
│ React Frontend     │
└─────────┬──────────┘
          │ REST / HTTPS
          ▼
┌────────────────────┐
│ NestJS Backend     │
│                    │
│ Business Logic     │
│ Authentication     │
│ Authorization      │
│ Validation         │
└──────┬─────────┬───┘
       │         │
       ▼         ▼
┌────────────┐  ┌────────────────────┐
│ PostgreSQL │  │ Notification Jobs  │
└────────────┘  └─────────┬──────────┘
                          │
                          ▼
                  ┌───────────────┐
                  │ Worker        │
                  └───────┬───────┘
                          │
                          ▼
                  ┌───────────────┐
                  │ Email Provider│
                  └───────────────┘
```

## 3. Frontend

The React frontend provides the user interface for students, faculty, and administrators.

The frontend communicates with the backend through REST APIs over HTTPS.

The frontend does not communicate directly with PostgreSQL.

## 4. Backend

The NestJS backend is the main business-logic layer.

Responsibilities include:

- Authentication
- Authorization
- Request validation
- Faculty management
- Availability management
- Appointment management
- Notification job creation
- API documentation
- Error handling
- Structured logging

The backend exposes versioned APIs under:

```text
/api/v1
```

## 5. Database

PostgreSQL is the system's source of truth.

TypeORM is used for database access and migrations.

Database schema changes are managed through migrations.

Automatic schema synchronization is disabled in production.

Critical appointment operations are performed using database transactions.

## 6. Authentication and Authorization

Authentication answers:

> Who is the user?

Authorization answers:

> What is the user allowed to do?

JWT-based authentication is used for authenticated API requests.

Role-based access control (RBAC) determines permissions for:

- Student
- Faculty
- Admin

Authorization is enforced by the backend.

Frontend restrictions are not considered security boundaries.

## 7. Appointment Integrity

Appointment conflicts must be protected at the database level.

The system must prevent concurrent requests from creating multiple active appointments for the same faculty time period.

Critical appointment operations use database transactions.

The database is therefore the final authority for appointment consistency.

## 8. Notifications

Notifications are handled asynchronously.

The backend creates durable notification jobs when an event requires a notification.

The independent worker processes pending jobs and communicates with the configured email provider.

This prevents email delivery from blocking normal API requests.

## 9. API Documentation

The REST API is documented using OpenAPI.

Swagger UI will be available during development to inspect and test API endpoints.

## 10. Logging

The backend uses structured logging.

Logs should contain useful operational information such as:

- Timestamp
- Log level
- Request or correlation ID
- Event
- Relevant resource ID
- Error information when applicable

Sensitive information such as passwords and authentication secrets must not be logged.

## 11. Deployment

Application components are containerized using Docker.

The planned deployment architecture consists of:

```text
Frontend
   │
   ▼
Managed Hosting

Backend
   │
   ▼
Managed Container Hosting
   │
   ├── PostgreSQL
   └── Notification Worker
```

Infrastructure will be managed using Terraform/OpenTofu.

CI/CD will automate testing, building, and deployment.

## 12. Architectural Principles

1. The frontend is untrusted.
2. The backend enforces business rules.
3. PostgreSQL is the source of truth.
4. Critical operations use transactions.
5. Database constraints protect data integrity.
6. Notifications are asynchronous.
7. APIs are versioned.
8. Secrets are stored through environment/configuration management.
9. Schema changes use migrations.
10. Services should remain independently deployable where practical.
