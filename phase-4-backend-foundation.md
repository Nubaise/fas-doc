# FAS — Phase 4: Backend Foundation

## 1. Phase Objective

Phase 4 establishes the backend foundation for the Faculty Appointment System.

The phase provides:

- NestJS backend structure
- PostgreSQL database connectivity
- TypeORM integration
- Database migration infrastructure
- Core database entities
- Database integrity constraints
- API foundation
- Environment configuration and validation
- Global request validation
- CORS configuration

Authentication and authorization are intentionally handled in Phase 5.

## 2. Backend Stack

The backend uses:

- NestJS
- TypeScript
- Node.js
- PostgreSQL
- Neon PostgreSQL
- TypeORM
- Zod
- REST API
- OpenAPI-compatible API structure

Database schema changes are managed through TypeORM migrations.

Automatic database schema synchronization is disabled.

## 3. Backend Module Structure

The backend is organized into feature modules.

Current modules:

```text
auth
users
departments
students
faculty
availability-schedules
availability-exceptions
appointments
notification-jobs
notifications
```

Each database-backed feature uses TypeORM's `forFeature()` integration for its entities.

## 4. Environment Configuration

NestJS `ConfigModule` is configured globally.

Environment variables are validated using Zod.

Required configuration:

```text
DATABASE_URL
PORT
```

`DATABASE_URL` must be a valid URL.

`PORT` must be a positive integer and defaults to `3000` when not provided.

Invalid configuration causes application startup to fail.

Sensitive environment files are excluded from Git.

A `.env.example` file documents the required environment variables without containing secrets.

## 5. Database Connection

The application connects to PostgreSQL through TypeORM.

The database URL is provided through the validated environment configuration.

The application uses:

```text
synchronize: false
```

Database schema changes are therefore performed through explicit migrations rather than automatic synchronization.

The application successfully connects to the Neon PostgreSQL database during startup.

## 6. TypeORM Migration Infrastructure

TypeORM migrations are configured through a dedicated DataSource:

```text
src/database/data-source.ts
```

Migration files are stored in:

```text
src/database/migrations/
```

The backend provides scripts for:

```text
migration:generate
migration:run
migration:revert
```

Migrations are executed through the TypeORM CLI using the CommonJS-compatible TypeScript runner.

The migration infrastructure was verified successfully against the Neon PostgreSQL database.

## 7. Database Entities Implemented

The following entities were implemented during Phase 4:

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

The corresponding TypeORM entities are organized within their feature modules.

## 8. Database Integrity

The database enforces the integrity rules defined in the database design.

### Availability schedules

The database enforces:

- `start_time` must be earlier than `end_time`.
- `slot_duration` must be one of `15`, `30`, `45`, or `60`.

### Availability exceptions

The database enforces:

- Whole-day exceptions require both time values to be `NULL`.
- Partial-day exceptions require both time values to be provided.
- Partial-day `start_time` must be earlier than `end_time`.

### Appointments

The database enforces:

- `start_time` must be earlier than `end_time`.
- `PENDING` and `CONFIRMED` appointments cannot overlap for the same faculty member.

Appointment overlap protection uses a PostgreSQL GiST exclusion constraint with the half-open range:

```text
[start_time, end_time)
```

The `btree_gist` PostgreSQL extension is used to support the exclusion constraint.

This provides database-level protection against concurrent appointment conflicts.

## 9. API Foundation

The backend API uses the global prefix:

```text
/api
```

URI-based API versioning is enabled.

The current API version structure is:

```text
/api/v1/...
```

This establishes a versioned API foundation for future endpoints.

## 10. Global Request Validation

Global request validation is enabled using NestJS's `StandardSchemaValidationPipe`.

Zod is used as the validation schema library.

Validation is configured globally so that future API endpoints can define request schemas consistently.

Transformation is enabled for validated request data.

## 11. CORS

CORS is enabled at the application level.

This provides the backend foundation required for the React frontend to communicate with the API.

More restrictive production CORS configuration can be introduced when deployment environments are defined.

## 12. Verification

The Phase 4 backend foundation was verified through the following checks.

### Build

```text
npm run build
```

The backend builds successfully.

### Application startup

```text
npm run start
```

The NestJS application starts successfully and connects to the Neon PostgreSQL database.

### Migration execution

```text
npm run migration:run
```

Migrations execute successfully.

Running the migration command again confirms that no pending migrations remain.

### Environment validation

Invalid database configuration was tested.

An invalid `DATABASE_URL` causes application startup to fail during configuration validation.

## 13. Phase 4 Git Milestones

Important implementation milestones include:

```text
26c8d4f  chore: initialize FAS backend
425874a  chore: remove Nest starter demo
90511c2  feat: add auth module
c3a2b93  feat: add users module
0f40f06  feat: add backend configuration foundation
8175a5e  feat: configure TypeORM database connection
f7367da  feat: add TypeORM migration infrastructure
287b9a9  feat: add departments entity and migration
a620864  feat: add students entity and migration
f5f307f  feat: add faculty entity and migration
a417ee6  feat: complete database schema foundation
545503a  feat: add API prefix and versioning
cc4373f  feat: add environment configuration validation
ccc3b1f  feat: add global request validation
3fafad6  feat: enable CORS
4213b41  feat: add database integrity checks
52670cb  refactor: use ConfigService for application port
```

The corresponding documentation update was committed separately in the documentation repository:

```text
dc07209  docs: document database integrity constraints
```

## 14. Phase 4 Completion Status

Phase 4 is complete.

The backend foundation, database schema, migration infrastructure, configuration validation, API foundation, and database integrity protection are implemented and verified.

The implementation and canonical documentation are synchronized.

The backend repository working tree is clean and synchronized with its remote repository.

The documentation repository working tree is clean and synchronized with its remote repository.

## 15. Phase Boundary

The following concerns are intentionally outside Phase 4:

- Password hashing implementation
- Login implementation
- JWT token generation
- JWT authentication guards
- Current-user handling
- Role-based authorization
- RBAC guards and decorators

These concerns belong to:

**Phase 5 — Authentication & Authorization**
