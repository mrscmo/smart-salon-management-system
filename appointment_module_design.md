# Appointment Management Module - API and Frontend Outline

This document details the backend API endpoints and frontend component structure for the Appointment Management module, building upon the `project_setup_plan.md`.

## I. Database Table Adjustments/Additions for Availability

To manage staff availability effectively, we'll refine the initial idea of storing it purely in a JSONB field in the `Staff` table. We'll introduce two dedicated tables:

1.  **`StaffRecurringAvailability`**: For general weekly schedules.
    *   `id` (SERIAL PRIMARY KEY)
    *   `staff_id` (INTEGER, REFERENCES `Users`(`id`) ON DELETE CASCADE)
    *   `day_of_week` (INTEGER NOT NULL, 0 for Sunday, 6 for Saturday) - CHECK (`day_of_week` BETWEEN 0 AND 6)
    *   `start_time` (TIME NOT NULL)
    *   `end_time` (TIME NOT NULL) - CHECK (`end_time` > `start_time`)
    *   `created_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP)
    *   `updated_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP)
    *   UNIQUE (`staff_id`, `day_of_week`, `start_time`, `end_time`)

2.  **`StaffAvailabilityOverrides`**: For specific date adjustments (days off, special hours).
    *   `id` (SERIAL PRIMARY KEY)
    *   `staff_id` (INTEGER, REFERENCES `Users`(`id`) ON DELETE CASCADE)
    *   `override_date` (DATE NOT NULL)
    *   `start_time` (TIME) - Nullable if the whole day is unavailable
    *   `end_time` (TIME) - Nullable if the whole day is unavailable, CHECK ( (`start_time` IS NULL AND `end_time` IS NULL) OR (`end_time` > `start_time`) )
    *   `is_unavailable` (BOOLEAN DEFAULT true) - If true, this period is off. If false, it could represent a special working period (overriding recurring).
    *   `notes` (TEXT, Optional, e.g., "Holiday", "Personal Leave")
    *   `created_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP)
    *   `updated_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP)
    *   UNIQUE (`staff_id`, `override_date`, `start_time`, `end_time`)

The `Appointments` table schema from `project_setup_plan.md` will be used as is.

## II. Backend API Development (Node.js/Express & PostgreSQL)

All endpoints should be protected by authentication middleware (`ensureAuthenticated`) and role-based access control (`ensureRole`) as appropriate. For example, staff managing their own availability, or admins managing any staff availability.

### 1. Staff Availability API Endpoints

*   **`POST /api/staff/{staffId}/availability/recurring`**
    *   Description: Define recurring weekly working hours for a staff member.
    *   Permissions: Admin, or Staff (for their own `staffId`).
    *   Request Body:
        ```json
        {
          "dayOfWeek": 1, // Monday
          "startTime": "09:00", // HH:MM format
          "endTime": "17:00"
        }
        ```
    *   Response:
        *   Success (201 Created): `{ "id": N, "staffId": X, "dayOfWeek": 1, "startTime": "09:00:00", "endTime": "17:00:00" }`
        *   Error (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 500 Internal Server Error).
    *   Logic: Creates a new entry in `StaffRecurringAvailability`.

*   **`DELETE /api/staff/{staffId}/availability/recurring/{recurringId}`**
    *   Description: Remove a recurring availability slot.
    *   Permissions: Admin, or Staff (for their own `staffId`).
    *   Response: Success (204 No Content) or error.

*   **`POST /api/staff/{staffId}/availability/override`**
    *   Description: Define a specific date override (e.g., day off, special working hours).
    *   Permissions: Admin, or Staff (for their own `staffId`).
    *   Request Body:
        ```json
        // For a day off (or specific unavailable block)
        {
          "overrideDate": "YYYY-MM-DD",
          "isUnavailable": true,
          "startTime": "10:00", // Optional, if entire day, can be null
          "endTime": "14:00",   // Optional
          "notes": "Doctor's appointment" // Optional
        }
        // For special working hours on a specific date
        {
          "overrideDate": "YYYY-MM-DD",
          "isUnavailable": false, // Explicitly working these hours
          "startTime": "10:00",
          "endTime": "16:00",
          "notes": "Special event coverage" // Optional
        }
        ```
    *   Response:
        *   Success (201 Created): `{ "id": N, "staffId": X, "overrideDate": "YYYY-MM-DD", ... }`
        *   Error.
    *   Logic: Creates a new entry in `StaffAvailabilityOverrides`.

*   **`DELETE /api/staff/{staffId}/availability/override/{overrideId}`**
    *   Description: Remove a specific date override.
    *   Permissions: Admin, or Staff (for their own `staffId`).
    *   Response: Success (204 No Content) or error.

*   **`GET /api/staff/{staffId}/availability?date=YYYY-MM-DD`**
    *   Description: Get calculated availability for a staff member on a specific date, considering recurring schedule and overrides.
    *   Permissions: Admin, Staff (for own ID), or Customer (when booking).
    *   Response:
        ```json
        // Example: Staff works 9-5, takes lunch 12-13, has an override making them unavailable 15-16
        {
          "date": "YYYY-MM-DD",
          "timeSlots": [
            { "startTime": "09:00", "endTime": "12:00" },
            { "startTime": "13:00", "endTime": "15:00" },
            { "startTime": "16:00", "endTime": "17:00" }
          ]
        }
        ```
    *   Logic:
        1.  Fetch recurring availability for the given `date`'s day of the week.
        2.  Fetch all overrides for that `date`.
        3.  Apply overrides:
            *   If `is_unavailable` is true, subtract that time from recurring slots.
            *   If `is_unavailable` is false, these are the *only* available slots for that day.
        4.  Return a list of continuous available blocks.

*   **`GET /api/availability?serviceId=X&date=YYYY-MM-DD`**
    *   Description: Get combined real-time availability slots for a given service on a specific date.
    *   Permissions: Customer, Staff, Admin.
    *   Response:
        ```json
        {
          "date": "YYYY-MM-DD",
          "serviceId": X,
          "availableSlots": [ // Sorted by time, then by staffId (or staff preference)
            { "staffId": Y, "staffName": "Jane Doe", "startTime": "09:00", "endTime": "09:30" }, // Assuming 30 min service
            { "staffId": Z, "staffName": "John Smith", "startTime": "09:00", "endTime": "09:30" },
            { "staffId": Y, "staffName": "Jane Doe", "startTime": "09:30", "endTime": "10:00" },
            // ... further slots, taking into account service duration and existing appointments
          ]
        }
        ```
    *   Logic:
        1.  Fetch the service details (`Services` table) to get `duration_minutes`.
        2.  Identify staff members qualified to perform the service (future enhancement: link `Services` to `Staff` via a join table `StaffServices`). For now, assume all staff can do all services or filter by a predefined list.
        3.  For each relevant staff member:
            a.  Get their availability for the given `date` (using logic similar to `GET /api/staff/{staffId}/availability`).
            b.  Fetch their existing appointments (`Appointments` table) for that `date`.
            c.  Subtract booked appointment times from their available blocks.
        4.  For each remaining available block for each staff member, calculate how many instances of the `service_duration` fit.
        5.  Compile a list of specific start times, paired with `staffId`.
        6.  Sort and return.

### 2. Service API Endpoints (CRUD)

Referencing `project_setup_plan.md`, these are standard CRUD operations.

*   **`POST /api/services`** (Admin)
    *   Request: `{ "name": "Haircut", "description": "...", "durationMinutes": 30, "price": 50.00, "isActive": true }`
    *   Response: (201 Created) Service object.
*   **`GET /api/services`** (Public/All Users)
    *   Response: (200 OK) `[ { "id": 1, "name": "Haircut", ... }, ... ]`
*   **`GET /api/services/{serviceId}`** (Public/All Users)
    *   Response: (200 OK) `{ "id": 1, "name": "Haircut", ... }` or (404 Not Found).
*   **`PUT /api/services/{serviceId}`** (Admin)
    *   Request: `{ "name": "Deluxe Haircut", "price": 60.00, ... }`
    *   Response: (200 OK) Updated service object or (404 Not Found).
*   **`DELETE /api/services/{serviceId}`** (Admin)
    *   Response: (204 No Content) or (404 Not Found).

### 3. Appointment Booking API Endpoints

*   **`POST /api/appointments/book`**
    *   Permissions: Customer (for self), Staff/Admin (can book for customers).
    *   Request:
        ```json
        {
          "customerId": X, // User ID of the customer
          "staffId": Y,
          "serviceId": Z,
          "startTime": "YYYY-MM-DD HH:MM" // Specific start time chosen by user
        }
        ```
    *   Response:
        *   Success (201 Created): `{ "id": A, "customerId": X, "staffId": Y, "serviceId": Z, "startTime": "...", "endTime": "...", "status": "scheduled", ... }`
        *   Error (400 Bad Request - e.g., slot not available, conflict, invalid input; 404 Not Found - e.g. customer/staff/service not found; 500).
    *   Logic:
        1.  Validate inputs.
        2.  Fetch service duration. Calculate `endTime`.
        3.  Verify staff availability for the exact `startTime` and `endTime` slot (critical check).
            *   This involves checking `StaffRecurringAvailability`, `StaffAvailabilityOverrides`, and existing `Appointments` for the chosen `staffId` to prevent double booking.
        4.  Create the appointment record with `status: 'scheduled'`.
        5.  Trigger basic confirmation email.

*   **`GET /api/appointments`**
    *   Permissions:
        *   Customer: Can see their own appointments (`?customerId=X`).
        *   Staff: Can see their own appointments (`?staffId=Y`).
        *   Admin: Can see all appointments.
    *   Query Parameters: `customerId` (integer), `staffId` (integer), `startDate` (YYYY-MM-DD), `endDate` (YYYY-MM-DD), `status` (string).
    *   Response: (200 OK) `[ { "id": A, ... }, ... ]`
    *   Logic: Fetch appointments from `Appointments` table, applying filters. Join with User (customer, staff) and Service tables for more details.

*   **`GET /api/appointments/{appointmentId}`**
    *   Permissions: Customer (own), Staff (own or if involved), Admin.
    *   Response: (200 OK) `{ "id": A, ... }` or (404 Not Found).

*   **`PUT /api/appointments/{appointmentId}`**
    *   Permissions: Admin, Staff (for their appointments, perhaps with limitations), Customer (for rescheduling their own, with policy limits).
    *   Request:
        ```json
        {
          // Only include fields to be updated
          "newStartTime": "YYYY-MM-DD HH:MM", // If rescheduling
          "newStaffId": W,                  // If changing staff
          "status": "confirmed"             // If confirming
        }
        ```
    *   Response: (200 OK) Updated appointment object or error.
    *   Logic:
        1.  Fetch existing appointment.
        2.  If `newStartTime` or `newStaffId` is provided, re-validate availability and check for conflicts for the new slot/staff.
        3.  Update appointment record.
        4.  Consider triggering notification emails for changes.

*   **`DELETE /api/appointments/{appointmentId}`** (Cancel appointment)
    *   Permissions: Admin, Staff (their appointments), Customer (their own, with policy limits e.g., not too close to appointment time).
    *   Response: (204 No Content) or error.
    *   Logic:
        1.  Fetch appointment.
        2.  Check cancellation policies (e.g., cannot cancel within 2 hours of start time - future).
        3.  Either hard delete or update status to `cancelled_by_customer` / `cancelled_by_salon`. Soft delete (status update) is generally preferred for history.
        4.  Consider triggering notification emails.

### 4. Automatic Confirmation Emails (Basic)

*   **Setup:**
    *   Use a library like `Nodemailer`.
    *   Configure with an SMTP transport (e.g., SendGrid, Mailgun, or even Gmail for development). Store credentials securely in `.env`.
*   **Trigger:**
    *   Within the `POST /api/appointments/book` success logic, after saving the appointment to the database.
*   **Content (Example):**
    *   To: Customer's email (from `Users` table)
    *   Subject: Your Appointment Confirmation - [Salon Name]
    *   Body:
        ```
        Hi [Customer First Name],

        Your appointment is confirmed!
        Service: [Service Name]
        Date: [Date]
        Time: [Start Time]
        With: [Staff First Name]

        Thank you for booking with [Salon Name].
        [Salon Address/Contact - Optional]
        ```
*   **Implementation Sketch (Conceptual):**
    ```javascript
    // services/emailService.js
    const nodemailer = require('nodemailer');
    const transporter = nodemailer.createTransport({ /* ... config ... */ });

    async function sendAppointmentConfirmation(customer, appointment, service, staff) {
      await transporter.sendMail({
        from: process.env.EMAIL_FROM,
        to: customer.email,
        subject: 'Your Appointment Confirmation - Salon XYZ',
        text: `Hi ${customer.firstName}, ... details ...` // Plain text
        // html: `<p>Hi ${customer.firstName}, ...</p>` // Or HTML
      });
    }
    module.exports = { sendAppointmentConfirmation };

    // controllers/appointmentController.js
    // ... inside bookAppointment handler ...
    // const { sendAppointmentConfirmation } = require('../services/emailService');
    // ... after successful DB save ...
    // await sendAppointmentConfirmation(customerDetails, newAppointment, serviceDetails, staffDetails);
    ```

## III. Frontend Outline (React)

### 1. Online Booking Components

*   **`ServiceList` Component:**
    *   Fetches and displays services from `GET /api/services`.
    *   Each service item could show name, price, duration.
    *   Allows user to select a service.
    *   Props: `onServiceSelect(serviceId)`
*   **`AvailabilityCalendar` Component:**
    *   Props: `serviceId`, `selectedDate`, `onDateChange(date)`, `onTimeSlotSelect(slot)`
    *   Displays a calendar (e.g., `react-calendar` or custom).
    *   When a date is selected (and `serviceId` is available):
        *   Fetches available slots using `GET /api/availability?serviceId=X&date=YYYY-MM-DD`.
        *   Displays available time slots (e.g., buttons for each `startTime`).
        *   May also show staff associated with each slot if multiple staff are available.
*   **`StaffSelector` (Optional/Conditional) Component:**
    *   Props: `serviceId`, `date`, `onStaffSelect(staffId)`
    *   Used if the user wants to pick a specific staff member *before* seeing time slots, or if the system allows choosing from available staff for a chosen slot.
    *   Could fetch staff who can perform `serviceId` and are generally available on `date`.
*   **`BookingForm` Component:**
    *   Props: `serviceId`, `selectedDate`, `selectedTimeSlot`, `selectedStaffId` (if applicable).
    *   Collects/displays customer information:
        *   If logged in, pre-fill from user context.
        *   If guest booking is allowed, fields for name, email, phone.
    *   Shows summary of booking: service, date, time, staff, price.
    *   On submit, calls `POST /api/appointments/book`.
    *   Handles loading states and error messages.
*   **`BookingConfirmation` Component:**
    *   Props: `appointmentDetails`
    *   Displays a success message with appointment details after successful booking.
    *   Options to add to calendar (ICS download link - future enhancement).

**Booking Flow (User Perspective):**
1.  User selects a Service (`ServiceList`).
2.  User picks a Date (and possibly time, or sees time slots) from `AvailabilityCalendar`.
    *   (Optional) User picks a Staff member.
3.  User reviews details and confirms (`BookingForm`).
4.  User sees `BookingConfirmation`.

### 2. Schedule Management Components (for Staff/Admin)

*   **`ScheduleCalendarView` Component:**
    *   The main dashboard for viewing and managing appointments.
    *   Uses a library like `react-big-calendar` or a custom implementation.
    *   Fetches appointments via `GET /api/appointments` with filters for date range, staff (if admin/multi-staff view), etc.
    *   Displays appointments as events on the calendar.
    *   Events are clickable to open `AppointmentModal`.
    *   **Drag-and-Drop:**
        *   Enabled if user has permissions.
        *   On drop, calls `PUT /api/appointments/{appointmentId}` with `newStartTime` and/or `newStaffId`.
        *   Requires careful validation on the backend.
*   **`AppointmentModal` Component:**
    *   Props: `appointmentData`, `isOpen`, `onClose`, `onEdit`, `onCancel`
    *   Displays detailed information about an appointment.
    *   Provides options to edit (opens `AppointmentFormModal`) or cancel the appointment (calls `DELETE /api/appointments/{appointmentId}`).
*   **`AppointmentFormModal` Component (for Create/Edit):**
    *   Props: `initialData` (for editing), `isOpen`, `onClose`, `onSave`
    *   A form for staff/admin to manually create or modify appointments.
    *   Fields: Customer selection (search/dropdown), service, staff, date, time.
    *   On save, calls `POST /api/appointments/book` (if new) or `PUT /api/appointments/{appointmentId}` (if editing).
    *   Needs validation for conflicts.
*   **`StaffAvailabilityManager` Component:**
    *   Interface for staff/admin to manage availability.
    *   **`RecurringAvailabilityForm`:**
        *   Select `dayOfWeek`, input `startTime`, `endTime`.
        *   Calls `POST /api/staff/{staffId}/availability/recurring`.
        *   Lists existing recurring slots with delete options (`DELETE /api/staff/{staffId}/availability/recurring/{recurringId}`).
    *   **`OverrideAvailabilityForm`:**
        *   Select `date`, `startTime` (optional), `endTime` (optional), `isUnavailable` checkbox.
        *   Calls `POST /api/staff/{staffId}/availability/override`.
        *   Lists existing overrides with delete options (`DELETE /api/staff/{staffId}/availability/override/{overrideId}`).

## IV. Booking Policy Considerations (Conceptual)

*   **Database:**
    *   In `Appointments` table:
        *   `booked_by_user_id` (FK to `Users`): Who made the booking (could be customer, staff, or admin).
        *   `created_at` (Timestamp): Already there, useful for tracking how far in advance bookings are made.
    *   Consider a `SalonSettings` or `BookingPolicies` table for future:
        *   `id` (PK)
        *   `salon_id` (FK, if multi-salon)
        *   `min_advance_booking_hours` (Integer, e.g., 1 hour)
        *   `max_advance_booking_days` (Integer, e.g., 90 days)
        *   `cancellation_cutoff_hours` (Integer, e.g., 2 hours before appointment)
        *   `allow_guest_booking` (Boolean)
*   **Backend Logic (Placeholders/Basic Implementation):**
    *   When booking (`POST /api/appointments/book`):
        *   Check if `startTime` is in the past (disallow).
        *   (Future) Check against `min_advance_booking_hours` and `max_advance_booking_days`.
    *   When cancelling (`DELETE /api/appointments/{appointmentId}`):
        *   (Future) Check against `cancellation_cutoff_hours`.
*   **Frontend:**
    *   The date/time pickers should disable past dates/times.
    *   Display policy information where relevant (e.g., "Cancellations must be made at least X hours in advance").

This document provides a detailed plan for the appointment management module. Implementation will follow these guidelines.
