# Phase 6 — Faculty & Department Management

## 1. Overview

Phase 6 implements faculty and department management on top of the existing FAS authentication, authorization, and database foundation.

The phase provides:

- Department management
- Faculty profile management
- Faculty directory and details
- Faculty ownership-based self-management
- Department assignment for faculty
- Role-based access control
- Validation and conflict handling

This phase does **not** implement availability, appointment slots, appointment booking, appointment lifecycle management, notifications, or frontend functionality. Those capabilities remain assigned to later phases.

---

## 2. Phase Objective

The objective of Phase 6 was to establish the backend management layer for:

1. Departments
2. Faculty profiles
3. Faculty-to-department relationships
4. Faculty profile ownership

The implementation builds on the existing:

- PostgreSQL database schema
- TypeORM entities
- JWT authentication
- RBAC
- global validation
- `/api/v1` API versioning
- NestJS feature-module architecture

---

## 3. Implemented Features

### 3.1 Department Management

Departments now support:

- List departments
- Retrieve a department by ID
- Create a department
- Update a department
- Delete a department

Department creation and updates enforce:

- Non-empty department name
- Non-empty department code
- Unique department name
- Unique department code

Department deletion is protected by the existing database relationships. If dependent records prevent deletion, the service returns a conflict response instead of allowing an unsafe deletion.

### 3.2 Faculty Management

Faculty profiles now support:

- List faculty
- Retrieve faculty by ID
- Create a faculty profile
- Update a faculty profile

Faculty creation verifies:

1. The referenced user exists.
2. The user has the `FACULTY` role.
3. The user is active.
4. The user does not already have a faculty profile.
5. The referenced department exists.
6. The employee number is unique.

### 3.3 Faculty Self-Management

Faculty members can update their own profile.

A faculty member may update:

- `firstName`
- `lastName`

A faculty member cannot update:

- `employeeNumber`
- `departmentId`
- another faculty member's profile

Ownership is enforced server-side in the faculty service.

### 3.4 Administrator Management

Administrators can create and update faculty profiles.

Administrators can manage:

- Employee number
- First name
- Last name
- Department assignment

Administrators can also create, update, and delete departments.

---

## 4. API Endpoints

All endpoints use the existing `/api/v1` API structure.

### 4.1 Departments

| Method | Endpoint                  | Access              |
| ------ | ------------------------- | ------------------- |
| GET    | `/api/v1/departments`     | Authenticated users |
| GET    | `/api/v1/departments/:id` | Authenticated users |
| POST   | `/api/v1/departments`     | Admin               |
| PATCH  | `/api/v1/departments/:id` | Admin               |
| DELETE | `/api/v1/departments/:id` | Admin               |

### 4.2 Faculty

| Method | Endpoint              | Access                         |
| ------ | --------------------- | ------------------------------ |
| GET    | `/api/v1/faculty`     | Authenticated users            |
| GET    | `/api/v1/faculty/:id` | Authenticated users            |
| POST   | `/api/v1/faculty`     | Admin                          |
| PATCH  | `/api/v1/faculty/:id` | Admin or owning faculty member |

---

## 5. Authorization Matrix

| Operation                      | Student | Faculty | Admin |
| ------------------------------ | ------: | ------: | ----: |
| View departments               |     Yes |     Yes |   Yes |
| Create department              |      No |      No |   Yes |
| Update department              |      No |      No |   Yes |
| Delete department              |      No |      No |   Yes |
| View faculty directory         |     Yes |     Yes |   Yes |
| View faculty details           |     Yes |     Yes |   Yes |
| Create faculty profile         |      No |      No |   Yes |
| Update own faculty profile     |      No |     Yes |   Yes |
| Update another faculty profile |      No |      No |   Yes |
| Change faculty department      |      No |      No |   Yes |
| Change faculty employee number |      No |      No |   Yes |

Authentication remains enforced by the existing global JWT guard, while role restrictions use the existing RBAC guard and `@Roles()` decorator.

---

## 6. Validation

Phase 6 continues the project's existing Zod validation convention.

### Department

Create:

- `name`: trimmed, non-empty string
- `code`: trimmed, non-empty string

Update:

- `name`: optional, trimmed, non-empty string
- `code`: optional, trimmed, non-empty string
- At least one field must be provided

### Faculty

Create:

- `userId`: UUID
- `employeeNumber`: trimmed, non-empty string
- `firstName`: trimmed, non-empty string
- `lastName`: trimmed, non-empty string
- `departmentId`: UUID

Update:

- `employeeNumber`: optional, trimmed, non-empty string
- `firstName`: optional, trimmed, non-empty string
- `lastName`: optional, trimmed, non-empty string
- `departmentId`: optional UUID
- At least one field must be provided

---

## 7. Business Rules

### Departments

- Department names must be unique.
- Department codes must be unique.
- Department deletion must respect dependent database records.
- Database uniqueness remains the final integrity boundary.

### Faculty

- A faculty profile must reference an existing user.
- The referenced user must have the `FACULTY` role.
- The referenced user must be active.
- A user can have only one faculty profile.
- Employee numbers must be unique.
- A faculty profile must reference an existing department.
- Faculty members can modify only their own permitted profile fields.
- Faculty ownership is checked server-side.
- Administrators are permitted to manage faculty records.

---

## 8. Error Handling

The implementation maps expected business failures to appropriate HTTP exceptions.

| Condition                                   | Response           |
| ------------------------------------------- | ------------------ |
| Invalid request body                        | `400 Bad Request`  |
| Missing authentication                      | `401 Unauthorized` |
| Insufficient role                           | `403 Forbidden`    |
| Faculty updates another profile             | `403 Forbidden`    |
| Faculty changes protected fields            | `403 Forbidden`    |
| Department/faculty/user does not exist      | `404 Not Found`    |
| Duplicate department name/code              | `409 Conflict`     |
| Duplicate employee number                   | `409 Conflict`     |
| Duplicate faculty profile                   | `409 Conflict`     |
| Inactive faculty user                       | `409 Conflict`     |
| Department deletion blocked by dependencies | `409 Conflict`     |

Database constraint violations are also handled at the service layer so that race-condition cases still produce appropriate conflict responses.

---

## 9. Module Structure

Phase 6 uses separate NestJS feature modules.

### Departments

```text
src/departments/
├── departments.controller.ts
├── departments.module.ts
├── departments.service.ts
├── departments.service.spec.ts
├── dto/
│   └── department.dto.ts
└── entities/
    └── department.entity.ts
```

### Faculty

```text
src/faculty/
├── faculty.controller.ts
├── faculty.module.ts
├── faculty.service.ts
├── faculty.service.spec.ts
├── dto/
│   └── faculty.dto.ts
└── entities/
    └── faculty.entity.ts
```

The faculty module reuses:

- `UsersModule`
- `DepartmentsModule`

This avoids duplicating user and department lookup logic.

---

## 10. User Service Extension

The existing `UsersService` was extended with:

```text
findById(id)
```

This allows faculty management to resolve a user by ID while keeping user repository access inside the users feature.

The existing:

```text
findByEmail(email)
```

continues to support authentication.

---

## 11. Testing

Phase 6 adds unit coverage for department and faculty management.

### Department tests

Department service tests cover:

- Listing departments
- Finding a department
- Missing department
- Successful creation
- Duplicate department name
- Duplicate department code
- Successful update
- Duplicate update conflicts
- Successful deletion
- Foreign-key deletion conflict

### Faculty tests

Faculty service tests cover:

- Listing faculty
- Finding faculty
- Missing faculty
- Successful faculty creation
- Missing user
- Incorrect user role
- Inactive user
- Duplicate faculty profile
- Duplicate employee number
- Administrator updates
- Faculty self-update
- Faculty ownership enforcement
- Protected employee number
- Protected department assignment
- Duplicate employee number during update
- Missing department
- Database unique-constraint handling

### Final Phase 6 test result

```text
Test Suites: 8 passed, 8 total
Tests:       41 passed, 41 total
Snapshots:   0 total
```

All existing authentication and authorization tests also continue to pass.

---

## 12. Quality Gates

The following checks were completed successfully:

### TypeScript

```text
npx tsc --noEmit
```

Result:

```text
Passed with 0 errors
```

### Tests

```text
npm test -- --runInBand
```

Result:

```text
8 test suites passed
41 tests passed
```

### Lint

```text
npm run lint
```

Result:

```text
Found 0 warnings and 0 errors.
```

### Build

```text
npm run build
```

Result:

```text
Passed
```

---

## 13. Architectural Compliance

Phase 6 follows the established FAS architecture.

- PostgreSQL remains the source of truth.
- TypeORM repositories are used for persistence.
- Feature-local NestJS modules encapsulate domain logic.
- JWT authentication remains globally enforced.
- RBAC remains globally enforced.
- Business authorization is enforced in the service layer.
- Zod remains the request-validation mechanism.
- API versioning remains URI-based.
- No database synchronization behavior was introduced.
- No future-phase appointment or availability logic was introduced.

---

## 14. Explicitly Deferred

The following are intentionally not part of Phase 6:

- Faculty availability schedules
- Availability exceptions
- Appointment slots
- Appointment booking
- Appointment conflict handling
- Appointment lifecycle
- Notifications
- Background notification worker
- Frontend faculty directory
- Frontend student UI
- Frontend faculty UI
- Admin UI

These capabilities remain assigned to later phases of the implementation plan.

---

## 15. Phase Completion Status

**Phase 6 — Faculty & Department Management: COMPLETE**

Implementation, testing, review, linting, and build verification have passed.

Next phase:

**Phase 7 — Availability & Slot System**
