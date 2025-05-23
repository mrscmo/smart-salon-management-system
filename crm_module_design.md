# Basic CRM Module - API and Frontend Design

This document outlines the backend APIs and frontend components for the Basic Customer Relationship Management (CRM) module. It builds upon `project_setup_plan.md` and `appointment_module_design.md`.

## I. Data Model Refinements (PostgreSQL)

Based on the requirements for CRM and refined appointment communication, the following adjustments and confirmations are made to the database schema:

1.  **`Customers` Table:**
    *   The initial `project_setup_plan.md` mentioned a `Customers` table that could be an extension of `Users`. For a more robust CRM, it's better to have distinct customer-specific information. However, for simplicity and to avoid data duplication, we will primarily use the `Users` table for core customer data if they have an account, and potentially a separate `GuestCustomers` table or a flag in `Users` if we need to store details of customers who haven't created an account (e.g., booked by phone).
    *   For this iteration, we'll assume customers who interact with the CRM will have a user account. The `Users` table already contains:
        *   `id` (Primary Key)
        *   `email` (String, Unique, Indexed)
        *   `first_name` (String)
        *   `last_name` (String)
        *   `phone_number` (String, Optional)
        *   `role` (Enum/String: 'admin', 'staff', **'customer'**)
        *   `created_at` (Timestamp)
        *   `updated_at` (Timestamp)
    *   **New Field/Table for Notes:** To store customer-specific notes and preferences, we can add a `notes` field to the `Users` table if the notes are general, or create a dedicated `CustomerNotes` table if we need more structured notes (e.g., multiple notes per customer with timestamps). For simplicity, let's start with a `notes` field in the `Users` table for users with the 'customer' role.
        *   **`Users` table modification:**
            *   Add `notes` (TEXT, Nullable) - Applicable when `role` = 'customer'.

2.  **`Appointments` Table:**
    *   The `appointment_module_design.md` already defined a comprehensive `Appointments` table.
    *   **Confirm/Add for Reminders:**
        *   `reminder_sent_at` (TIMESTAMP WITH TIME ZONE, Nullable): To track when a reminder email was sent for this appointment. This is crucial for the reminder system.
        *   `confirmation_email_sent_at` (TIMESTAMP WITH TIME ZONE, Nullable): To track when the initial booking confirmation email was sent.

    *Updated `Appointments` Table structure snippet:*
    ```sql
    CREATE TABLE Appointments (
        id SERIAL PRIMARY KEY,
        customer_id INTEGER REFERENCES Users(id) NOT NULL, -- Assuming customer is always a registered user
        staff_id INTEGER REFERENCES Users(id) NOT NULL,    -- Staff is also a registered user
        service_id INTEGER REFERENCES Services(id) NOT NULL,
        start_time TIMESTAMP WITH TIME ZONE NOT NULL,
        end_time TIMESTAMP WITH TIME ZONE NOT NULL,
        status VARCHAR(50) DEFAULT 'scheduled' CHECK (status IN ('scheduled', 'confirmed', 'completed', 'cancelled_by_customer', 'cancelled_by_salon', 'no_show')),
        notes TEXT, -- Notes specific to this appointment
        confirmation_email_sent_at TIMESTAMP WITH TIME ZONE, -- New
        reminder_sent_at TIMESTAMP WITH TIME ZONE,       -- New
        created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
        -- Potentially: booked_by_user_id INTEGER REFERENCES Users(id) -- if staff books for customer
    );
    ```

## II. Backend API Development (Node.js/Express & PostgreSQL)

Authentication (`ensureAuthenticated`) and Authorization (`ensureRole`) middleware will be applied as appropriate.

### 1. Customer Profile API Endpoints

Since customer data is primarily within the `Users` table for users with the 'customer' role, these APIs will largely interact with the `Users` table, filtering by role.

*   **`GET /api/customers`**
    *   Description: List all customers (users with 'customer' role).
    *   Permissions: Admin, Staff.
    *   Query Parameters: `search` (for name/email), `page`, `limit`.
    *   Response (200 OK):
        ```json
        {
          "customers": [
            {
              "id": 1,
              "firstName": "John",
              "lastName": "Doe",
              "email": "john.doe@example.com",
              "phoneNumber": "123-456-7890",
              "createdAt": "YYYY-MM-DDTHH:mm:ss.sssZ"
            }
            // ... more customers
          ],
          "totalPages": 5,
          "currentPage": 1
        }
        ```
    *   Logic: Retrieves users where `role = 'customer'`.

*   **`GET /api/customers/{customerId}`**
    *   Description: Get detailed profile for a specific customer.
    *   Permissions: Admin, Staff (for any customer), Customer (for their own ID).
    *   Response (200 OK):
        ```json
        {
          "id": 1,
          "firstName": "John",
          "lastName": "Doe",
          "email": "john.doe@example.com",
          "phoneNumber": "123-456-7890",
          "notes": "Prefers evening appointments. Allergic to lavender.",
          "createdAt": "YYYY-MM-DDTHH:mm:ss.sssZ",
          "updatedAt": "YYYY-MM-DDTHH:mm:ss.sssZ",
          "appointmentSummary": [ // Example, could be more detailed or paginated
            { "appointmentId": 101, "serviceName": "Haircut", "date": "YYYY-MM-DD", "status": "completed" },
            { "appointmentId": 105, "serviceName": "Manicure", "date": "YYYY-MM-DD", "status": "scheduled" }
          ]
        }
        ```
        or (404 Not Found).
    *   Logic:
        1.  Fetch user from `Users` table where `id = {customerId}` and `role = 'customer'`.
        2.  Fetch recent/summary appointment data from `Appointments` table for this `customerId`.

*   **`PUT /api/customers/{customerId}`**
    *   Description: Update customer profile information.
    *   Permissions: Admin, Staff (for any customer), Customer (for their own ID, possibly limited fields).
    *   Request Body:
        ```json
        {
          "firstName": "Johnathan", // Optional
          "lastName": "Doe",        // Optional
          "phoneNumber": "987-654-3210", // Optional
          "notes": "Prefers morning appointments now." // Optional
          // "email": "new.email@example.com" // Updating email (login) is more complex, might need separate process with verification
        }
        ```
    *   Response (200 OK): Updated customer object (as in `GET /api/customers/{customerId}`) or (404 Not Found).
    *   Logic:
        1.  Updates fields in the `Users` table for the given `customerId`.
        2.  If `email` is updated, it's a sensitive operation. It might require re-verification. For this iteration, we might restrict email changes by customers directly or handle it as a special admin task. Staff/Admin changes to email should be handled carefully.

*   **`POST /api/customers`** (Manual Customer Creation by Staff/Admin)
    *   Description: Create a new customer profile. This implies creating a new entry in the `Users` table with `role = 'customer'`.
    *   Permissions: Admin, Staff.
    *   Request Body:
        ```json
        {
          "firstName": "Jane",
          "lastName": "Smith",
          "email": "jane.smith@example.com", // Required, will be their login if they choose to set a password
          "phoneNumber": "555-123-4567", // Optional
          "notes": "New customer, referred by Alice.", // Optional
          "sendInvite": true // Optional: Flag to send an email inviting them to set a password and access their account
        }
        ```
    *   Response (201 Created): New customer object or (400 Bad Request - e.g., email already exists).
    *   Logic:
        1.  Check if email already exists in `Users`.
        2.  Create a new user in `Users` with `role = 'customer'`. Password can be null initially or a temporary one.
        3.  If `sendInvite` is true, trigger an email with a link to set/reset their password.

*   **`GET /api/customers/{customerId}/appointments`**
    *   Description: Get all appointments for a specific customer.
    *   Permissions: Admin, Staff, Customer (for their own ID).
    *   This endpoint is effectively an alias or direct use of the existing `GET /api/appointments?customerId={customerId}` defined in `appointment_module_design.md`. No new backend logic is strictly needed if the existing endpoint provides sufficient detail and filtering.
    *   Response: (200 OK) Array of appointment objects.

### 2. Automated Appointment Reminders (Email)

*   **Refined Logic & Scheduler:**
    *   A scheduled task (cron job) will run periodically (e.g., every hour or every few hours).
    *   **Technology:** `node-cron` is a suitable library for managing scheduled tasks within the Node.js application.
        ```javascript
        // conceptual code in a scheduler service file
        const cron = require('node-cron');
        const appointmentService = require('./services/appointmentService'); // a service to get appointments
        const emailService = require('./services/emailService'); // your existing email service

        // Schedule a task to run, e.g., every hour
        cron.schedule('0 * * * *', async () => {
          console.log('Running hourly check for appointment reminders...');
          try {
            const upcomingAppointments = await appointmentService.getAppointmentsNeedingReminders();
            for (const appt of upcomingAppointments) {
              const customer = await userService.getUserById(appt.customer_id); // Fetch customer details
              const service = await serviceService.getServiceById(appt.service_id); // Fetch service details
              const staff = await userService.getUserById(appt.staff_id); // Fetch staff details

              await emailService.sendAppointmentReminder(customer, appt, service, staff);
              await appointmentService.markReminderSent(appt.id);
            }
          } catch (error) {
            console.error('Error sending appointment reminders:', error);
          }
        });
        ```
*   **Database Query for Scheduler (`getAppointmentsNeedingReminders` logic):**
    *   The service method would query the `Appointments` table:
        *   `status` is 'scheduled' or 'confirmed'.
        *   `reminder_sent_at` IS NULL.
        *   `start_time` is within the reminder window (e.g., `start_time >= NOW() + interval '23 hours'` AND `start_time <= NOW() + interval '25 hours'` for a 24-hour reminder, if the cron runs hourly). Adjust window based on cron frequency to avoid missing appointments or sending multiple reminders. A simpler approach: `start_time` is between `NOW()` and `NOW() + interval '1 day'`.
*   **Tracking Reminder Status:**
    *   The `Appointments` table has `reminder_sent_at` (TIMESTAMP, nullable).
    *   After successfully sending a reminder email, the `markReminderSent(appointmentId)` service method updates this field to the current timestamp.
*   **Email Content (Reminder):**
    *   To: Customer's email.
    *   Subject: Reminder: Your Appointment at [Salon Name] on [Date].
    *   Body:
        ```
        Hi [Customer First Name],

        This is a friendly reminder about your upcoming appointment:

        Service: [Service Name]
        Date: [Date] (e.g., Tomorrow, or Full Date)
        Time: [Start Time]
        With: [Staff First Name]

        [Optional: Link to view/manage appointment - if feature exists]
        [Optional: Salon Address & Contact Number]
        [Optional: Cancellation/Reschedule Policy Snippet]

        We look forward to seeing you!

        Thanks,
        The [Salon Name] Team
        ```

## III. Frontend Outline (React)

### 1. Customer Profile Components (for Admin/Staff View in a `/admin/customers` section)

*   **`CustomerListPage` Component:**
    *   Route: `/admin/customers`
    *   Fetches customers using `GET /api/customers`.
    *   Displays a `CustomerTable` component.
    *   Includes search input and pagination controls.
    *   Button to navigate to a "New Customer" form/page.
*   **`CustomerTable` Component:**
    *   Props: `customers` (array), `onSelectCustomer(customerId)`
    *   Renders a table with columns: Name, Email, Phone, Joined Date.
    *   Each row is clickable, navigating to `CustomerDetailPage` or invoking `onSelectCustomer`.
*   **`CustomerDetailPage` Component:**
    *   Route: `/admin/customers/{customerId}`
    *   Fetches detailed customer data using `GET /api/customers/{customerId}`.
    *   Displays:
        *   `CustomerInfoCard`: Shows basic info (name, email, phone).
        *   `CustomerEditForm`: Inline editing or modal form to update details (uses `PUT /api/customers/{customerId}`). Includes fields for `firstName`, `lastName`, `phoneNumber`, `notes`.
        *   `CustomerAppointmentHistory`: Fetches and lists appointments using `GET /api/customers/{customerId}/appointments` (or the general appointment endpoint with customer filter). Displays key appointment details (service, date, staff, status).
*   **`NewCustomerPage` / `NewCustomerFormModal` Component:**
    *   Route: `/admin/customers/new` or modal triggered from `CustomerListPage`.
    *   Form with fields: First Name, Last Name, Email, Phone Number, Notes.
    *   Checkbox: "Send account invitation email".
    *   On submit, calls `POST /api/customers`. Handles success (e.g., redirect to detail page) and errors (e.g., email exists).

### 2. Customer-Facing Profile Components (for logged-in Customers in a `/profile` section)

*   **`MyProfilePage` Component:**
    *   Route: `/profile` or `/account/profile`.
    *   Fetches the logged-in user's data (can use a generic `/api/users/me` endpoint or `GET /api/customers/{auth.userId}`).
    *   Displays:
        *   `EditableProfileInfo`: Shows `firstName`, `lastName`, `email` (display only or link to change email process), `phoneNumber`. Allows updating `firstName`, `lastName`, `phoneNumber`. Uses `PUT /api/customers/{auth.userId}`.
        *   `MyUpcomingAppointments`: Lists appointments where `customerId` is the logged-in user's ID and `status` is 'scheduled' or 'confirmed'. Fetches from `GET /api/appointments?customerId={auth.userId}&status=upcoming`.
        *   `MyPastAppointments`: Lists appointments where `customerId` is the logged-in user's ID and `status` is 'completed', 'cancelled', etc. Fetches from `GET /api/appointments?customerId={auth.userId}&status=past`.
        *   (Each appointment could link to a detailed view or offer options like "Book Again" or "Cancel" if applicable based on `appointment_module_design.md`.)

## IV. Deliverables Summary for this Document (`crm_module_design.md`)

*   **Database Schema Updates:**
    *   `Users` table: Added `notes` (TEXT, Nullable).
    *   `Appointments` table: Added `confirmation_email_sent_at` (TIMESTAMP, Nullable) and `reminder_sent_at` (TIMESTAMP, Nullable).
*   **Defined API Endpoints for Basic CRM:**
    *   `GET /api/customers`
    *   `GET /api/customers/{customerId}`
    *   `PUT /api/customers/{customerId}`
    *   `POST /api/customers`
    *   `GET /api/customers/{customerId}/appointments` (leveraging existing appointment API)
    *   Request/response structures are outlined for each.
*   **Design for Automated Reminder System:**
    *   Scheduler: `node-cron`.
    *   Logic Flow: Cron job queries for appointments needing reminders, fetches details, sends email via `emailService`, and updates `reminder_sent_at` status in `Appointments` table.
    *   Database Change: `reminder_sent_at` field in `Appointments`.
*   **Outline of React Components for Customer Profile Management:**
    *   Admin/Staff Views: `CustomerListPage`, `CustomerTable`, `CustomerDetailPage`, `NewCustomerPage/Modal`.
    *   Customer Views: `MyProfilePage` with sub-components for info and appointment lists.

This design provides a foundational plan for the CRM module, focusing on customer data management and enhancing appointment communications.
