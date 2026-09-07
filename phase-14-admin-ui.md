# Phase 14 — Admin UI

## Overview

Phase 14 implements the administrator-facing web interface for the Faculty Appointment System (FAS).

The Admin UI provides authenticated administrators with a dedicated portal for managing faculty, departments, faculty availability, and appointments.

The implementation uses the existing backend APIs and follows the frontend architecture established during Phase 11, while reusing the UI and data-management patterns established during the Student UI and Faculty UI phases.

## Goals

- Build the admin dashboard.
- Build faculty management interfaces.
- Build department management interfaces.
- Build faculty availability management interfaces.
- Build appointment administration interfaces.
- Support faculty onboarding through the existing faculty workflow.
- Support bulk faculty onboarding through CSV import.
- Provide faculty search and management actions.
- Provide department CRUD operations.
- Provide appointment search, filtering, and read-only details.
- Protect admin routes through the existing authentication and authorization flow.
- Provide appropriate loading, empty, error, conflict, and success states.
- Reuse the existing frontend API client, query layer, routing, and UI architecture.
- Maintain responsive and accessible interfaces.
- Keep backend business rules and authorization as the source of truth.

## Admin Dashboard

The admin dashboard provides administrators with navigation to the main administrative areas of FAS.

It includes access to:

- Faculty management.
- Department management.
- Faculty availability management.
- Appointment management.

The dashboard is protected by the existing admin authentication and authorization flow.

## Faculty Management

The faculty management interface allows administrators to view and manage faculty records.

The interface provides:

- Faculty listing.
- Faculty search.
- Faculty detail access.
- Faculty creation.
- Faculty editing.
- Faculty availability access.
- Faculty onboarding.
- Faculty management actions through the existing backend APIs.

Faculty records display the information returned by the backend.

The backend remains responsible for faculty validation, authorization, uniqueness constraints, and business rules.

## Bulk Faculty Onboarding

Phase 14 adds bulk faculty onboarding through CSV import.

The CSV import supports the following fields:

- Email.
- Initial password.
- Employee number.
- First name.
- Last name.
- Department ID.

The interface provides:

- CSV file selection.
- CSV parsing.
- Import preview.
- Department resolution using department ID, code, or name.
- Validation before submission.
- Bulk submission to the backend.
- Successful import results.
- Per-row failure reporting.

The preview does not display faculty passwords.

Bulk onboarding reports failed rows with:

- Row number.
- Email.
- Employee number.
- Failure message.

Each faculty record is processed independently by the backend so that a failure in one row does not prevent other valid rows from being created.

The backend remains responsible for final validation, duplicate detection, authorization, password hashing, department validation, and transactional faculty creation.

## Department Management

The department management interface provides administrators with department CRUD functionality.

The interface supports:

- Department listing.
- Department creation.
- Department editing.
- Department deletion.

Department records use the existing backend department API.

When a department cannot be deleted because dependent records exist, the backend conflict response is presented through an appropriate UI conflict state.

The frontend does not independently determine whether a department can be deleted.

## Faculty Availability Management

Administrators can access availability management for individual faculty members.

The availability interface uses the existing availability schedule and exception APIs.

It provides access to:

- Weekly availability schedules.
- Availability schedule configuration.
- Availability exceptions.

Existing backend rules continue to control:

- Schedule validation.
- Overlap protection.
- Slot duration.
- Availability conflicts.
- Appointment slot generation.

The Admin UI does not introduce separate availability business rules.

## Appointment Management

The appointment management interface provides administrators with a read-only view of appointments.

It includes:

- Appointment listing.
- Appointment search.
- Appointment filtering.
- Appointment detail view.

The appointment list allows administrators to locate appointments using the available search and filtering functionality.

The appointment detail page displays the appointment information returned by the backend.

The Admin UI does not provide appointment lifecycle actions from the administrator appointment detail interface. Appointment lifecycle rules remain controlled by the backend and the appropriate student/faculty workflows.

## Routing

Admin routes are protected through the existing authentication and authorization structure.

Admin UI routes include:

- Admin dashboard.
- Admin faculty listing.
- Admin faculty creation.
- Admin faculty details.
- Admin faculty editing.
- Admin faculty availability.
- Admin department management.
- Admin appointment listing.
- Admin appointment details.

The implementation reuses the existing frontend router, protected routes, role-based authorization, and authentication context.

## API Integration

The Admin UI uses the existing centralized frontend API client.

API integrations include:

- Faculty data.
- Faculty onboarding.
- Bulk faculty onboarding.
- Department data.
- Availability schedules.
- Availability exceptions.
- Appointment data.

Bulk faculty onboarding uses the backend endpoint:

```text
POST /api/v1/faculty/bulk-onboard
```
