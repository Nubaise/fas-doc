# FAS — Phase 5: Authentication & Authorization

## 1. Phase Objective

Phase 5 establishes authentication and authorization for the Faculty Appointment System.

The phase provides:

- Password verification using Argon2
- User lookup through a dedicated UsersService
- Login API
- JWT access-token generation
- JWT authentication
- Current-user handling
- Role-based access control
- Authorization guards and decorators
- Authentication and authorization testing
- Security configuration and validation

The system supports the following roles:

```text
STUDENT
FACULTY
ADMIN
```

Refresh-token infrastructure is intentionally outside Phase 5.

## 2. Authentication Stack

The backend authentication implementation uses:

- NestJS
- `@nestjs/jwt`
- `@nestjs/passport`
- Passport
- `passport-jwt`
- Argon2
- PostgreSQL user records
- Zod environment validation

Passwords are never stored or compared as plaintext.

Password verification uses Argon2.

JWT access tokens are used to authenticate API requests.

## 3. Backend Module Structure

Authentication is implemented within the `auth` module.

The authentication module contains:

```text
auth
├── auth.controller.ts
├── auth.module.ts
├── auth.service.ts
├── decorators/
│   ├── current-user.decorator.ts
│   ├── public.decorator.ts
│   └── roles.decorator.ts
├── dto/
│   └── login.dto.ts
├── guards/
│   ├── jwt-auth.guard.ts
│   └── roles.guard.ts
├── strategies/
│   └── jwt.strategy.ts
└── types/
    └── role.type.ts
```

The `users` module provides a `UsersService` abstraction for retrieving user records required by authentication.

## 4. User Lookup

Authentication uses a dedicated `UsersService`.

The service provides user lookup by email:

```text
findByEmail(email)
```

The authentication layer does not access the TypeORM repository directly.

This keeps user persistence concerns separated from authentication logic.

## 5. Password Verification

User passwords are represented by the database field:

```text
password_hash
```

The authentication service verifies the supplied password against the stored Argon2 hash.

The verification flow is:

```text
Login request
      |
      v
Validate request schema
      |
      v
Find user by email
      |
      v
Check account is active
      |
      v
Verify password with Argon2
      |
      v
Generate JWT
```

Invalid credentials produce the same generic authentication response:

```text
401 Unauthorized
Invalid credentials
```

The implementation does not expose whether the email exists, whether the account is inactive, or whether the password was incorrect.

## 6. Login API

The authentication endpoint is:

```text
POST /api/v1/auth/login
```

Request body:

```json
{
  "email": "user@example.com",
  "password": "password"
}
```

The request is validated using the login Zod schema.

A successful login returns:

```json
{
  "accessToken": "<jwt>"
}
```

The access token is subsequently supplied through the HTTP authorization header:

```text
Authorization: Bearer <jwt>
```

## 7. JWT Design

JWT access tokens contain only the minimum information required by the authorization layer.

Current claims:

```json
{
  "sub": "<user-uuid>",
  "role": "STUDENT"
}
```

`sub` identifies the authenticated user.

`role` identifies the user's application role.

Sensitive user information is not included in the JWT payload.

JWT expiration is configured through the environment variable:

```text
JWT_EXPIRES_IN
```

The current default is:

```text
15m
```

JWT signing uses the secret configured through:

```text
JWT_SECRET
```

The JWT secret is never committed to the repository.

## 8. JWT Authentication Guard

Protected API routes use a JWT authentication guard.

The guard extracts the JWT from:

```text
Authorization: Bearer <jwt>
```

The JWT strategy validates:

- Token signature
- Token expiration
- JWT payload structure

A valid JWT produces the authenticated user context:

```text
{
  id,
  role
}
```

Invalid or missing authentication results in:

```text
401 Unauthorized
```

## 9. Public Routes

Authentication is enforced globally using a NestJS global guard.

Routes that intentionally do not require authentication are explicitly marked with:

```text
@Public()
```

The login endpoint is public because users must be able to authenticate before receiving a JWT.

This approach makes authentication the default for future endpoints rather than requiring every new controller to remember to add an authentication guard.

## 10. Current User Context

The `@CurrentUser()` decorator provides the authenticated user's minimal identity context to controllers.

The current authenticated context is:

```text
{
  id: string,
  role: Role
}
```

Controllers can therefore access the authenticated user's identity without directly reading the Passport request object.

## 11. Role-Based Access Control

FAS uses role-based access control.

Supported roles are:

```text
STUDENT
FACULTY
ADMIN
```

Routes can declare required roles using:

```text
@Roles(...)
```

The `RolesGuard` checks the authenticated user's role against the roles required by the route.

Authorization is enforced by the backend.

Frontend role checks are not considered a security boundary.

## 12. Authorization Semantics

The authentication and authorization layers use the following HTTP semantics.

### 401 Unauthorized

Returned when authentication has not been successfully established.

Examples:

- Missing JWT
- Invalid JWT
- Expired JWT
- Invalid login credentials

### 403 Forbidden

Returned when authentication succeeds but the authenticated user does not have permission to access the requested resource.

Example:

```text
Authenticated STUDENT
        |
        v
ADMIN-only endpoint
        |
        v
403 Forbidden
```

## 13. Guard Execution

Authentication and authorization are registered as global guards.

The effective request flow is:

```text
HTTP request
    |
    v
JwtAuthGuard
    |
    +---- public route --> continue
    |
    +---- valid JWT ----> authenticated user
    |
    +---- invalid/missing JWT --> 401
    |
    v
RolesGuard
    |
    +---- no role restriction --> continue
    |
    +---- required role matches --> continue
    |
    +---- required role does not match --> 403
    |
    v
Controller
```

Authentication therefore occurs before role-based authorization.

## 14. Environment Configuration

Authentication introduces the following environment variables:

```text
JWT_SECRET
JWT_EXPIRES_IN
```

The configuration is validated using Zod.

`JWT_SECRET` must contain at least 32 characters.

`JWT_EXPIRES_IN` is configurable and defaults to:

```text
15m
```

The example environment file documents these variables without containing an actual secret.

## 15. ESM Backend Configuration

During Phase 5, the backend was migrated to an ECMAScript Modules (ESM) configuration.

The backend uses:

```text
"type": "module"
```

and TypeScript NodeNext module resolution.

The TypeScript configuration uses:

```text
module: nodenext
moduleResolution: nodenext
```

Local TypeScript imports use explicit `.js` extensions so that the emitted ESM JavaScript resolves correctly.

Jest is configured for ESM execution using Node's VM modules support.

The test environment uses `ts-jest` with ESM support.

The ESM configuration was adopted to align the backend with the current NestJS package model.

## 16. TypeORM Migration CLI

Because the backend now uses ESM, the TypeORM migration scripts use the ESM TypeScript runner:

```text
typeorm-ts-node-esm
```

The dedicated TypeORM DataSource remains responsible for migration configuration.

The migration infrastructure was verified after the ESM migration.

Running:

```text
npm run migration:run
```

successfully connects to the PostgreSQL database and reports that no migrations are pending when the database is already current.

## 17. Security Decisions

Phase 5 establishes the following security decisions:

- Passwords are never stored as plaintext.
- Argon2 is used for password verification.
- JWT secrets are provided through environment configuration.
- JWT secrets are not committed to Git.
- JWT payloads contain only the user identifier and role.
- Authentication is enforced globally by default.
- Public routes must be explicitly marked.
- Authorization is enforced on the backend.
- Incorrect roles return `403 Forbidden`.
- Authentication failures return `401 Unauthorized`.
- Invalid login attempts use a generic `Invalid credentials` response.
- Refresh tokens are not implemented in this phase.

JWT role information is evaluated from the authenticated token. Role changes therefore take effect for newly issued tokens; existing tokens remain valid until they expire.

The configured short-lived access-token lifetime reduces the duration of this window.

## 18. Testing

Authentication and authorization were verified through unit and end-to-end tests.

### TypeScript checks

```text
npx tsc --noEmit
```

Passed.

```text
npx tsc --project test/tsconfig.json --noEmit
```

Passed.

### Build

```text
npm run build
```

Passed.

### Unit tests

```text
npm test -- --runInBand
```

Result:

```text
6 test suites passed
14 tests passed
```

The unit tests cover:

- Credential validation
- JWT generation
- JWT strategy validation
- JWT authentication guard
- Current-user decorator
- Roles decorator
- Roles guard

### End-to-end tests

```text
npm run test:e2e
```

Result:

```text
1 test suite passed
6 tests passed
```

The E2E tests verify:

1. Public routes work without a JWT.
2. Protected routes reject requests without a JWT.
3. Invalid JWTs are rejected.
4. Admin-only routes reject unauthenticated requests.
5. A STUDENT cannot access an ADMIN-only route.
6. An authenticated STUDENT can access a protected route.

The E2E authorization tests use a test-only controller because production application controllers currently do not contain a representative role-protected business endpoint.

## 19. Verification Summary

Phase 5 verification completed successfully.

```text
TypeScript source check       PASS
Test TypeScript check         PASS
Production build              PASS
Unit tests                    14/14 PASS
End-to-end tests              6/6 PASS
```

The backend authentication and authorization foundation is therefore implemented and verified.

## 20. Known Warning

The E2E test run reports a PostgreSQL SSL warning concerning future changes to `pg-connection-string` SSL-mode semantics.

The warning does not currently cause test or application failure.

The current behavior should be reviewed as part of a future infrastructure/security configuration pass rather than changed during Phase 5 without a deliberate decision.

The Node VM Modules experimental warning is expected from the ESM Jest execution configuration.

## 21. Phase 5 Git Milestones

The Phase 5 implementation is currently prepared for its final implementation commit after documentation review.

The documentation repository will record the corresponding Phase 5 documentation milestone separately.

## 22. Phase 5 Completion Status

Phase 5 implementation and verification are complete.

Authentication and authorization are implemented using:

```text
Argon2
JWT
Passport
Global authentication guard
Global RBAC guard
@Public()
@CurrentUser()
@Roles()
```

The backend build, TypeScript checks, unit tests, and E2E tests all pass.

Canonical documentation is being synchronized with the implementation before the phase is committed.

## 23. Phase Boundary

The following concerns are intentionally outside Phase 5:

- Refresh-token infrastructure
- Appointment-specific authorization policies
- Detailed student/faculty resource ownership rules
- Production deployment security configuration
- Production CORS restriction
- Email notification implementation

These concerns belong to later phases where their business and deployment requirements are defined.
