# Appointment Management Module Enhancements - Design Document

This document details the design for enhancements to the Appointment Management module, including calendar synchronization, waitlist management, no-show tracking, recurring appointments, and SMS notifications. It references and extends `appointment_module_design.md`.

## I. Calendar Synchronization (e.g., Google Calendar - Initial Integration)

This feature allows staff members to connect their Google Calendar to the salon system, enabling salon appointments to be automatically added to their personal calendar.

### 1. Database Schema Changes (PostgreSQL)

*   **`Users` Table (Staff):**
    *   Add fields to store OAuth tokens for Google Calendar integration:
        *   `google_calendar_access_token` (TEXT, Nullable)
        *   `google_calendar_refresh_token` (TEXT, Nullable)
        *   `google_calendar_token_expiry_date` (TIMESTAMP WITH TIME ZONE, Nullable)
        *   `google_calendar_sync_enabled` (BOOLEAN, Default: false)
    *SQL conceptual additions:*
    ```sql
    ALTER TABLE Users
    ADD COLUMN google_calendar_access_token TEXT,
    ADD COLUMN google_calendar_refresh_token TEXT,
    ADD COLUMN google_calendar_token_expiry_date TIMESTAMP WITH TIME ZONE,
    ADD COLUMN google_calendar_sync_enabled BOOLEAN DEFAULT false;
    ```

*   **`Appointments` Table:**
    *   Add a field to store the Google Calendar event ID for synced appointments:
        *   `google_calendar_event_id` (TEXT, Nullable)
    *SQL conceptual additions:*
    ```sql
    ALTER TABLE Appointments
    ADD COLUMN google_calendar_event_id TEXT;
    ```

### 2. OAuth Integration (Backend - Node.js)

*   **Library:** `googleapis` (official Google API Node.js client library).
*   **Configuration:**
    *   Store Google Cloud Project `client_id` and `client_secret` securely in environment variables.
    *   Define `redirect_uri` (e.g., `https://api.yourdomain.com/auth/google/callback`).
*   **API Endpoints:**
    *   **`GET /api/staff/{staffId}/calendar/google/auth/initiate`**
        *   Permissions: Staff (for their own `staffId`), Admin.
        *   Logic:
            1.  Generates the Google OAuth 2.0 consent URL with required scopes (e.g., `https://www.googleapis.com/auth/calendar.events`).
            2.  Includes `state` parameter for security (e.g., containing `staffId`).
            3.  Redirects the user to this URL.
    *   **`GET /auth/google/callback`** (This is the `redirect_uri` registered with Google)
        *   Logic:
            1.  Handles the callback from Google after user consent.
            2.  Receives `code` and `state`. Verifies `state`.
            3.  Exchanges the `code` for tokens (access token, refresh token, expiry date) using `googleapis`.
            4.  Retrieves `staffId` from the verified `state`.
            5.  Securely stores `access_token`, `refresh_token`, and `expiry_date` in the `Users` table for the respective `staffId`. Sets `google_calendar_sync_enabled = true`.
            6.  Redirects user to a frontend page indicating success or failure.
    *   **`POST /api/staff/{staffId}/calendar/google/auth/disconnect`**
        *   Permissions: Staff (for own `staffId`), Admin.
        *   Logic: Clears `google_calendar_access_token`, `google_calendar_refresh_token`, `google_calendar_token_expiry_date`, and sets `google_calendar_sync_enabled = false` for the staff member. Optionally, attempts to revoke the token with Google.

### 3. Synchronization Logic (Backend)

*   **`CalendarSyncService`:**
    *   **Token Management:**
        *   Helper function to get a valid access token for a staff member, refreshing it using the `refresh_token` if it's expired. Updates stored token and expiry.
    *   **`createGoogleCalendarEvent(appointmentId, staffId)`:**
        *   Called after a new appointment is created for a synced staff member.
        *   Fetches appointment details and staff's Google Calendar tokens.
        *   Constructs event data (summary, description, start time, end time).
        *   Uses `googleapis` to insert the event into the staff's primary Google Calendar.
        *   Stores the returned `google_calendar_event_id` in the `Appointments` table.
    *   **`updateGoogleCalendarEvent(appointmentId, staffId)`:**
        *   Called after an appointment is updated for a synced staff member.
        *   Fetches the `google_calendar_event_id`. If it exists, updates the event in Google Calendar.
    *   **`deleteGoogleCalendarEvent(appointmentId, staffId)`:**
        *   Called after an appointment is cancelled/deleted for a synced staff member.
        *   Fetches the `google_calendar_event_id`. If it exists, deletes the event from Google Calendar.
*   **Trigger Points:** Integrate calls to `CalendarSyncService` methods within the appointment creation, update, and deletion logic in `AppointmentController` or `AppointmentService`.
*   **Initial Sync (Optional/Future):** A one-time job to sync existing appointments for a staff member when they first enable sync.
*   **Two-Way Sync:** Not in scope for initial integration due to complexity (would involve webhook subscriptions to Google Calendar changes and logic to reconcile with salon system availability).

### 4. UI (Frontend)

*   **Staff Profile Page (`/profile/settings` or similar):**
    *   "Google Calendar Sync" section.
    *   Button: "Connect Google Calendar". If not connected, initiates the OAuth flow by redirecting to `GET /api/staff/{staffId}/calendar/google/auth/initiate`.
    *   Status Display: Shows "Connected" or "Not Connected".
    *   Button: "Disconnect Google Calendar". Calls `POST /api/staff/{staffId}/calendar/google/auth/disconnect`.

## II. Waitlist Management

### 1. Data Model (PostgreSQL)

*   **`WaitlistEntries` Table:**
    *   `id` (SERIAL PRIMARY KEY)
    *   `customer_id` (INTEGER NOT NULL, REFERENCES `Users`(`id`) ON DELETE CASCADE)
    *   `service_id` (INTEGER NOT NULL, REFERENCES `Services`(`id`) ON DELETE CASCADE)
    *   `staff_preference_id` (INTEGER, REFERENCES `Users`(`id`) ON DELETE SET NULL, Nullable): Optional preferred staff.
    *   `requested_date_start` (DATE NOT NULL): Preferred start date for availability.
    *   `requested_date_end` (DATE NOT NULL): Preferred end date for availability.
    *   `time_preferences` (TEXT, Nullable): E.g., "morning", "afternoon", "any". Could be JSONB for more structure: `{"morning": true, "evening": false}`.
    *   `notes` (TEXT, Nullable): Customer notes.
    *   `status` (TEXT NOT NULL DEFAULT 'pending', CHECK (`status` IN ('pending', 'notified', 'booked', 'expired', 'cancelled_by_customer'))): Status of the waitlist entry.
    *   `created_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP)
    *   `updated_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP)

    *SQL for `WaitlistEntries` Table:*
    ```sql
    CREATE TABLE WaitlistEntries (
        id SERIAL PRIMARY KEY,
        customer_id INTEGER NOT NULL REFERENCES Users(id) ON DELETE CASCADE,
        service_id INTEGER NOT NULL REFERENCES Services(id) ON DELETE CASCADE,
        staff_preference_id INTEGER REFERENCES Users(id) ON DELETE SET NULL,
        requested_date_start DATE NOT NULL,
        requested_date_end DATE NOT NULL CHECK (requested_date_end >= requested_date_start),
        time_preferences TEXT, -- Consider JSONB for structured preferences
        notes TEXT,
        status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'notified', 'booked', 'expired', 'cancelled_by_customer')),
        created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
    );
    CREATE INDEX idx_waitlist_customer ON WaitlistEntries(customer_id);
    CREATE INDEX idx_waitlist_service ON WaitlistEntries(service_id);
    CREATE INDEX idx_waitlist_status ON WaitlistEntries(status);
    CREATE INDEX idx_waitlist_requested_dates ON WaitlistEntries(requested_date_start, requested_date_end);
    ```

### 2. API Endpoints (Backend)

*   **`POST /api/waitlist`**
    *   Permissions: Customer (for self), Staff/Admin (can add for customer).
    *   Request: `{ customerId, serviceId, staffPreferenceId (optional), requestedDateStart, requestedDateEnd, timePreferences (optional), notes (optional) }`
    *   Response: (201 Created) New waitlist entry object.
*   **`GET /api/waitlist`**
    *   Permissions: Admin, Staff.
    *   Query Params: `serviceId`, `staffId`, `date`, `status`.
    *   Response: (200 OK) Array of waitlist entries.
*   **`PUT /api/waitlist/{entryId}`** (Generic update, including status)
    *   Permissions: Admin, Staff.
    *   Request: `{ status (e.g., "notified", "booked", "expired", "cancelled_by_customer"), notes (optional) }`
    *   Response: (200 OK) Updated waitlist entry.
*   **`DELETE /api/waitlist/{entryId}`** (Effectively sets status to 'cancelled' or hard delete based on policy)
    *   Permissions: Admin, Staff, Customer (own entry if status is 'pending').
    *   Response: (204 No Content) or (200 OK with updated object if soft delete).

### 3. Logic (Backend - `WaitlistService`)

*   **Slot Availability Check:**
    *   When an appointment is cancelled/rescheduled:
        1.  Identify the opened slot (service, staff, date, time).
        2.  Query `WaitlistEntries` for matching criteria:
            *   `service_id` matches.
            *   `status` is 'pending'.
            *   `requested_date_start` <= opened slot date <= `requested_date_end`.
            *   `staff_preference_id` matches or is null.
            *   `time_preferences` match (requires parsing/logic).
        3.  Order potential matches (e.g., by `created_at` oldest first).
*   **Notification (Initial - Manual/Internal):**
    *   If matches found, create an internal notification for staff/admin (e.g., a dashboard alert or email to salon).
    *   "Customer [Name] on waitlist for [Service] might fit the slot opened by cancellation of appointment [ID]."
    *   Staff can then manually contact customer and use `PUT /api/waitlist/{entryId}` to update status to `notified`. If customer accepts, staff books appointment and updates status to `booked`.

### 4. UI (Frontend - Admin/Staff in `/admin/waitlist`)

*   **`WaitlistPage` Component:**
    *   Displays `WaitlistTable` with filters (service, staff, date range, status).
    *   Button to "Add to Waitlist" (opens `WaitlistFormModal`).
*   **`WaitlistTable` Component:**
    *   Lists entries: Customer, Service, Requested Dates, Staff Pref., Status, Actions (Edit, Notify, Book, Cancel).
*   **`WaitlistFormModal` Component:** For creating/editing waitlist entries.
*   **Booking Flow Integration:** If customer attempts to book and no slots are available, offer "Add to Waitlist" button.
*   **Cancellation Flow Integration:** After cancelling an appointment, if potential waitlist matches exist, show an alert/suggestion to staff.

## III. No-Show Tracking

### 1. Data Model (PostgreSQL)

*   **`Appointments` Table:**
    *   The `status` field (from `appointment_module_design.md`) should be extended or ensured to include 'No-Show'.
        *   `status` (TEXT, CHECK (`status` IN ('scheduled', 'confirmed', 'completed', 'cancelled_by_customer', 'cancelled_by_salon', 'no_show', ...)))
    *   No separate table is needed for basic tracking. A `no_show_count` on `Users` could be a denormalization for performance if frequently accessed, but can be calculated from appointments.

### 2. API Endpoints (Backend)

*   **`PUT /api/appointments/{appointmentId}`**
    *   Request body can include `status: "No-Show"`.
    *   Permissions: Staff, Admin.
    *   Logic:
        1.  Update appointment status.
        2.  (Optional/Future) Increment a `no_show_count` on the `Users` table for the customer if denormalizing.
        3.  (Optional/Future) Trigger policy actions (e.g., warning email, future booking restrictions if multiple no-shows).

### 3. UI (Frontend - Admin/Staff)

*   **Appointment Modal/Details View:**
    *   Add a button "Mark as No-Show". This calls `PUT /api/appointments/{appointmentId}` with `status: "No-Show"`.
    *   Confirmation prompt before marking.
*   **Customer Profile Page (`/admin/customers/{customerId}`):**
    *   Display a "No-Show History" section or a "No-Show Count".
    *   Fetched by querying customer's appointments with `status = 'No-Show'`.

## IV. Recurring Appointment Setup

### 1. Data Model (PostgreSQL)

*   **`RecurringAppointments` Table:**
    *   `id` (SERIAL PRIMARY KEY)
    *   `customer_id` (INTEGER NOT NULL, REFERENCES `Users`(`id`))
    *   `staff_id` (INTEGER NOT NULL, REFERENCES `Users`(`id`))
    *   `service_id` (INTEGER NOT NULL, REFERENCES `Services`(`id`))
    *   `start_datetime` (TIMESTAMP WITH TIME ZONE NOT NULL): Start date and time of the first appointment in the series.
    *   `end_date` (DATE, Nullable): Date after which no more appointments should be generated.
    *   `recurrence_rule` (TEXT NOT NULL): Example: `FREQ=WEEKLY;INTERVAL=1;BYDAY=MO` (every Monday), `FREQ=MONTHLY;BYMONTHDAY=15` (15th of every month). Store as an iCalendar RRULE string.
    *   `next_occurrence_date` (DATE): Optimization to know when to next generate instances.
    *   `is_active` (BOOLEAN DEFAULT true): To deactivate the entire series.
    *   `created_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP)
    *   `updated_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP)

*   **`Appointments` Table:**
    *   Add `recurring_appointment_id` (INTEGER, Nullable, REFERENCES `RecurringAppointments`(`id`) ON DELETE SET NULL): Links an individual appointment instance to its parent recurring series. `ON DELETE SET NULL` means if the series definition is deleted, the past individual appointments remain but are orphaned (or handle this via specific logic).

    *SQL for `RecurringAppointments` Table:*
    ```sql
    CREATE TABLE RecurringAppointments (
        id SERIAL PRIMARY KEY,
        customer_id INTEGER NOT NULL REFERENCES Users(id) ON DELETE CASCADE,
        staff_id INTEGER NOT NULL REFERENCES Users(id) ON DELETE CASCADE, -- Or ON DELETE SET NULL / RESTRICT based on policy
        service_id INTEGER NOT NULL REFERENCES Services(id) ON DELETE CASCADE, -- Or ON DELETE SET NULL / RESTRICT
        start_datetime TIMESTAMP WITH TIME ZONE NOT NULL,
        end_date DATE,
        recurrence_rule TEXT NOT NULL, -- Store iCalendar RRULE string
        next_occurrence_date DATE, -- For generation job
        is_active BOOLEAN DEFAULT TRUE,
        created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
        CONSTRAINT recurring_end_date_check CHECK (end_date IS NULL OR end_date >= DATE(start_datetime))
    );
    ALTER TABLE Appointments ADD COLUMN recurring_appointment_id INTEGER REFERENCES RecurringAppointments(id) ON DELETE SET NULL;
    CREATE INDEX idx_appointments_recurring_id ON Appointments(recurring_appointment_id);
    ```

### 2. API Endpoints (Backend)

*   **`POST /api/appointments/recurring`**
    *   Permissions: Staff, Admin.
    *   Request: `{ customerId, staffId, serviceId, startDatetime, endDate (optional), recurrenceRule (RRULE string) }`
    *   Logic:
        1.  Validate inputs. Parse `recurrenceRule`.
        2.  Create `RecurringAppointments` record.
        3.  Generate initial set of `Appointments` (e.g., for next 3-6 months or until `endDate`) based on the rule. Link them with `recurring_appointment_id`. Store these as individual appointments.
        4.  Set `next_occurrence_date` on the parent.
    *   Response: (201 Created) The `RecurringAppointments` object and perhaps the first few generated instances.
*   **`GET /api/recurringappointments`**
    *   Permissions: Admin, Staff.
    *   Query: `customerId`, `staffId`.
    *   Response: List of `RecurringAppointments` series.
*   **`PUT /api/recurringappointments/{recurringId}`** (Modify Series) - *Complex*
    *   Permissions: Admin, Staff.
    *   Request: `{ staffId (optional), serviceId (optional), recurrenceRule (optional), endDate (optional), isActive (optional) }`
    *   Logic:
        *   If fundamental aspects change (rule, staff, service):
            *   Option 1 (Simpler): Cancel all future, non-completed `Appointments` linked to this series. Create new `Appointments` based on the updated rule from today/next valid date.
            *   Option 2 (Complex): Attempt to update individual future appointments if possible, or delete and regenerate.
        *   Changing `endDate` or `isActive` is simpler.
    *   Response: Updated `RecurringAppointments` object.
*   **`DELETE /api/recurringappointments/{recurringId}`** (Cancel Series)
    *   Permissions: Admin, Staff.
    *   Logic:
        1.  Set `RecurringAppointments.is_active = false` (soft delete of series definition).
        2.  Delete all future `Appointments` linked via `recurring_appointment_id` that are not yet 'completed' or 'cancelled'.
    *   Response: (204 No Content) or success message.
*   **Individual Instance Modification:**
    *   Use existing `PUT /api/appointments/{appointmentId}` and `DELETE /api/appointments/{appointmentId}`.
    *   If an appointment is part of a series (`recurring_appointment_id` is not null):
        *   Frontend should warn: "This is part of a recurring series. Modify only this instance, or the entire series?"
        *   If only this instance: Update/delete it. It becomes an exception.
        *   If entire series: Redirect to series management UI.

### 3. Logic (Backend - `RecurringAppointmentService`)

*   **Instance Generation:**
    *   Use a library like `rrule.js` to interpret RRULE strings and generate dates.
    *   A scheduled job (e.g., daily/weekly) checks `RecurringAppointments` where `next_occurrence_date` is near and `is_active = true`.
    *   Generates new `Appointments` instances up to a certain horizon (e.g., 6 months ahead or `end_date`), ensuring no conflicts. Updates `next_occurrence_date`.
*   **Conflict Handling:** When generating instances, check for staff availability and existing appointments. If a conflict arises for an instance, it could be skipped, or flagged for manual resolution.

### 4. UI (Frontend - Admin/Staff)

*   **Appointment Form:**
    *   Checkbox: "Make Recurring".
    *   If checked, show recurrence options:
        *   Frequency: Daily, Weekly, Monthly, Custom (to help build RRULE).
        *   Interval: Every X days/weeks/months.
        *   Days of week (if weekly). Day of month (if monthly).
        *   End Date (optional).
*   **`RecurringSeriesListPage` Component:**
    *   Lists active recurring series.
    *   Options to view instances, edit series, or cancel series.

## V. SMS Notifications (Integration with Twilio)

### 1. Configuration & Setup (Backend)

*   **Provider Choice:** Twilio (popular, robust).
*   **Credentials:** Store Twilio `Account SID`, `Auth Token`, and `Twilio Phone Number` in environment variables.
*   **Library:** `twilio` Node.js SDK.

### 2. `SMSService` (Backend)

*   `constructor(apiKey, apiSecret, senderPhoneNumber)`
*   `async sendMessage(recipientPhoneNumber, messageBody)`:
    *   Uses Twilio client to send SMS.
    *   Handles success/failure, logs the attempt.

### 3. Data Model (PostgreSQL)

*   **`Users` Table:**
    *   Ensure `phone_number` field exists (from `project_setup_plan.md`).
    *   Add `phone_number_verified` (BOOLEAN, Default: false).
    *   Add `sms_notifications_enabled` (BOOLEAN, Default: false) - Customer preference.
*   **`SMSLogs` Table (Optional but Recommended):**
    *   `id` (SERIAL PRIMARY KEY)
    *   `recipient_number` (TEXT NOT NULL)
    *   `message_body` (TEXT NOT NULL)
    *   `status` (TEXT NOT NULL, e.g., "sent", "failed", "delivered" - if using webhooks for status updates)
    *   `provider_message_id` (TEXT, Nullable): ID from Twilio.
    *   `error_message` (TEXT, Nullable)
    *   `sent_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP)

    *SQL for `SMSLogs` Table:*
    ```sql
    CREATE TABLE SMSLogs (
        id SERIAL PRIMARY KEY,
        recipient_number TEXT NOT NULL,
        message_body TEXT NOT NULL,
        status TEXT NOT NULL,
        provider_message_id TEXT,
        error_message TEXT,
        sent_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
    );
    ```

### 4. Integration Points (Backend)

*   Modify existing services (e.g., `AppointmentService`, `CRMService` for reminders, `WaitlistService`):
    *   **Appointment Confirmation:** If `customer.sms_notifications_enabled` and `customer.phone_number_verified`.
    *   **Appointment Reminders (Cron Job):** Similar check.
    *   **Waitlist Notification (Manual Trigger by Staff):** Staff can trigger an SMS to a waitlisted customer if their number is available. (Requires a new endpoint or modification to `PUT /api/waitlist/{entryId}/notify` to also trigger SMS).
*   **Phone Verification (Future):** Implement a one-time code via SMS for users to verify their phone number, setting `phone_number_verified = true`.

### 5. UI (Frontend)

*   **Customer Profile Page (`/profile/settings`):**
    *   Field for `phone_number`.
    *   Button "Verify Phone" (if not verified).
    *   Checkbox "Enable SMS Notifications".
*   **Admin Settings (Basic SMS Templates):**
    *   A simple UI for admins to view/edit the default text for appointment confirmations and reminders sent via SMS. (e.g., `/admin/settings/sms-templates`). These templates would be stored in a config file or a simple database table.

## VI. Deliverables Summary

*   This document (`appointment_enhancements_design.md`) outlining:
    *   Database schema changes/additions for all five enhancements.
    *   API endpoint definitions (routes, request/response, permissions).
    *   Key backend logic considerations.
    *   Frontend UI component outlines and user flow descriptions.
    *   Choice of Google Calendar API and Twilio for SMS, with basic integration plans.

This design provides a comprehensive plan for these advanced appointment management features.This subtask is a comprehensive design task covering five major enhancements to the Appointment Management module. I have reviewed the detailed requirements and will now submit the `appointment_enhancements_design.md` file I've just created, which addresses each of these points.

The file includes:

1.  **Calendar Synchronization (Google Calendar):**
    *   Database schema changes to `Users` (for OAuth tokens) and `Appointments` (for Google Calendar event ID).
    *   OAuth 2.0 flow with backend API endpoints for initiation, callback, and disconnect.
    *   Synchronization logic for creating, updating, and deleting events in Google Calendar when salon appointments change.
    *   Frontend UI considerations for staff to connect/disconnect their Google Calendar.

2.  **Waitlist Management:**
    *   Database schema for `WaitlistEntries` table.
    *   API endpoints for adding, viewing, updating (notifying, booking), and removing waitlist entries.
    *   Backend logic for matching opened slots with waitlist entries and initial notification ideas.
    *   Frontend UI for admin/staff to manage waitlists and for customers to join a waitlist.

3.  **No-Show Tracking:**
    *   Confirmation of using the `Appointments.status` field (extended to include "No-Show").
    *   API modifications to `PUT /api/appointments/{appointmentId}` to allow setting this status.
    *   Frontend UI for staff to mark no-shows and for viewing no-show history on customer profiles.

4.  **Recurring Appointment Setup:**
    *   Database schema for `RecurringAppointments` table (with RRULE storage) and linkage to the `Appointments` table.
    *   API endpoints for creating, viewing, modifying, and cancelling recurring appointment series.
    *   Backend logic for generating individual appointment instances from a recurrence rule, including a scheduled job for ongoing generation.
    *   Frontend UI for setting up recurrence patterns and managing series.

5.  **SMS Notifications (Twilio Integration):**
    *   Backend configuration for Twilio credentials.
    *   Design of an `SMSService` for sending messages.
    *   Database schema changes to `Users` (for phone verification and SMS preference) and a new `SMSLogs` table.
    *   Integration points for sending SMS for confirmations, reminders, and potentially waitlist notifications.
    *   Frontend UI for customer SMS preferences and basic admin template management.

All these sections detail database changes, API definitions, key backend logic, and frontend UI outlines as requested.
