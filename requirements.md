# FAS — Requirements

## 1. Purpose

FAS (Faculty Appointment System) allows students to request appointments with faculty based on faculty-defined availability.

## 2. User Roles

### Student

Students can:

- Log in
- View faculty
- View faculty availability
- Select an available time slot
- Submit an appointment request with a reason
- View appointment status
- Receive notifications

Students cannot:

- Cancel appointments
- Reschedule appointments
- Modify submitted appointment details
- Approve or reject appointments
- Manage faculty availability

### Faculty

Faculty can:

- Log in
- Manage their profile
- Define and manage availability
- Configure appointment duration
- Add availability exceptions
- View appointment requests
- Accept or reject requests
- View appointments
- Cancel appointments
- Reschedule appointments
- Receive notifications

### Admin

Admins can:

- Log in
- Manage users
- Manage faculty
- Manage departments
- Oversee appointments
- Manage system configuration
- View system activity

## 3. Appointment Flow

```text
Student Request
      │
      ▼
   PENDING
   /     \
  ▼       ▼
ACCEPT   REJECT
  │         │
  ▼         ▼
CONFIRMED  REJECTED
```

Faculty can subsequently cancel or reschedule applicable appointments.

## 4. Appointment Rules

- Students request appointments from system-generated available slots.
- Students cannot enter arbitrary appointment times.
- The default appointment duration is 30 minutes.
- Faculty can configure their standard duration as 15, 30, 45, or 60 minutes.
- A slot cannot have multiple active appointment requests or appointments.
- Slot conflicts must be protected at the database level.
- Rejected requests release the requested slot.
- Students cannot cancel or reschedule appointments.
- Faculty can cancel or reschedule appointments.
- Rescheduling must preserve appointment history.
- Faculty availability is based on recurring schedules.
- Temporary unavailable periods are represented as availability exceptions.

## 5. Appointment Statuses

```text
PENDING
CONFIRMED
REJECTED
CANCELLED
COMPLETED
```

`NO_SHOW` may be considered in a future version.

## 6. Availability

Faculty availability consists of recurring schedules containing:

- Day of week
- Start time
- End time
- Slot duration
- Active/inactive state

Temporary exceptions can make a whole day or a specific time range unavailable.

The system does not permanently store generated slots. Available slots are derived from recurring availability and existing appointments.

## 7. Notifications

The system should notify users when:

- An appointment request is submitted
- A request is accepted
- A request is rejected
- An appointment is cancelled
- An appointment is rescheduled

Notifications are processed asynchronously through durable notification jobs.

## 8. Security Requirements

- Authentication is JWT-based.
- Authorization is role-based.
- Backend authorization must not rely on frontend restrictions.
- Passwords must never be stored in plaintext.
- Critical appointment operations must use database transactions.
- Database constraints must protect appointment integrity.
