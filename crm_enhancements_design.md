# CRM Module Enhancements - Design Document

This document details the design for enhancements to the Customer Relationship Management (CRM) module, including allergy/sensitivity tracking, photo documentation for customer profiles, and implementing actual birthday/anniversary reminder notifications. It builds upon `crm_module_design.md` and `project_setup_plan.md`.

## I. Allergy/Sensitivity Tracking

### 1. Data Model (PostgreSQL)

*   **`Users` Table (for `role = 'customer'`):**
    *   As per `crm_module_design.md`, customer-specific data is primarily stored in the `Users` table for users with the 'customer' role.
    *   Add a new field:
        *   `allergies_sensitivities` (JSONB, Nullable): This allows storing structured data, which is more flexible than plain TEXT.
            *   Example JSON structure:
                ```json
                [
                  {"allergen": "PPD", "reaction_type": "Skin Rash", "severity": "High", "notes": "Avoid all dark hair dyes."},
                  {"allergen": "Lavender Oil", "reaction_type": "Headache", "severity": "Medium", "notes": "Scent sensitivity."}
                ]
                ```
            *   If no entries, this field will be `null` or an empty array `[]`.

    *SQL conceptual addition to `Users` table:*
    ```sql
    ALTER TABLE Users
    ADD COLUMN allergies_sensitivities JSONB;
    ```

### 2. API Endpoints (Backend - Node.js/Express)

*   **`GET /api/customers/{customerId}`:**
    *   (Modify existing endpoint from `crm_module_design.md`)
    *   Permissions: Admin, Staff, Customer (for their own ID).
    *   Response: Ensure the `allergies_sensitivities` field (JSONB) is included in the customer object.
        ```json
        // Example snippet of response
        {
          // ... other customer fields
          "allergies_sensitivities": [
            {"allergen": "PPD", "reaction_type": "Skin Rash", "severity": "High", "notes": "Avoid all dark hair dyes."}
          ]
          // ...
        }
        ```

*   **`PUT /api/customers/{customerId}`:**
    *   (Modify existing endpoint from `crm_module_design.md`)
    *   Permissions: Admin, Staff (for any customer), Customer (for their own ID).
    *   Request Body: Allow `allergies_sensitivities` (JSONB array) to be part of the updatable fields.
        ```json
        // Example snippet of request body
        {
          "allergies_sensitivities": [
            {"allergen": "PPD", "reaction_type": "Skin Rash", "severity": "High", "notes": "Avoid all dark hair dyes."},
            {"allergen": "New Allergen", "reaction_type": "Itching", "severity": "Low", "notes": "Discovered recently."}
          ]
          // ... other fields to update
        }
        ```
    *   Logic: The backend should validate the structure of the JSONB array if specific fields within each object are mandatory (e.g., `allergen`, `reaction_type`).

### 3. UI (Frontend - Admin/Staff React Components)

*   **Customer Detail View/Edit Form (`CustomerDetailPage` from `crm_module_design.md`):**
    *   Add a dedicated section "Allergies & Sensitivities".
    *   **Viewing:** Display existing entries in a readable format (e.g., a list or table).
    *   **Editing:**
        *   Allow adding new entries (fields for allergen, reaction type, severity, notes).
        *   Allow editing existing entries.
        *   Allow deleting entries.
        *   This UI would manage the JSON array structure, which is then sent to the `PUT /api/customers/{customerId}` endpoint.
*   **Appointment Booking/View:**
    *   When a customer with known allergies/sensitivities is selected or viewed in the context of an appointment:
        *   Display a prominent, easily noticeable alert or icon (e.g., a red exclamation mark next to the customer's name).
        *   Hovering over or clicking this alert should show a summary of their allergies/sensitivities.
        *   This information should be clearly visible to staff before starting any service.

## II. Photo Documentation (Customer Profiles)

### 1. Data Model (PostgreSQL)

*   **`CustomerPhotos` Table:**
    *   `id` (SERIAL PRIMARY KEY)
    *   `customer_id` (INTEGER NOT NULL, REFERENCES `Users`(`id`) ON DELETE CASCADE)
    *   `photo_url` (TEXT NOT NULL): URL to the stored photo.
    *   `description` (TEXT, Nullable): Optional description (e.g., "Before haircut March 2024").
    *   `uploaded_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)
    *   `uploaded_by_staff_id` (INTEGER, REFERENCES `Users`(`id`) ON DELETE SET NULL, Nullable): Staff member who uploaded the photo.
    *   `service_id` (INTEGER, REFERENCES `Services`(`id`) ON DELETE SET NULL, Nullable): Optional link to a specific service.
    *   `appointment_id` (INTEGER, REFERENCES `Appointments`(`id`) ON DELETE SET NULL, Nullable): Optional link to a specific appointment.

    *SQL for `CustomerPhotos` Table:*
    ```sql
    CREATE TABLE CustomerPhotos (
        id SERIAL PRIMARY KEY,
        customer_id INTEGER NOT NULL REFERENCES Users(id) ON DELETE CASCADE,
        photo_url TEXT NOT NULL,
        description TEXT,
        uploaded_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
        uploaded_by_staff_id INTEGER REFERENCES Users(id) ON DELETE SET NULL,
        service_id INTEGER REFERENCES Services(id) ON DELETE SET NULL,
        appointment_id INTEGER REFERENCES Appointments(id) ON DELETE SET NULL,
        CONSTRAINT fk_customer_photo_user FOREIGN KEY (customer_id) REFERENCES Users(id) ON DELETE CASCADE,
        CONSTRAINT fk_customer_photo_staff FOREIGN KEY (uploaded_by_staff_id) REFERENCES Users(id) ON DELETE SET NULL,
        CONSTRAINT fk_customer_photo_service FOREIGN KEY (service_id) REFERENCES Services(id) ON DELETE SET NULL,
        CONSTRAINT fk_customer_photo_appointment FOREIGN KEY (appointment_id) REFERENCES Appointments(id) ON DELETE SET NULL
    );
    CREATE INDEX idx_customerphotos_customer_id ON CustomerPhotos(customer_id);
    ```

### 2. Storage Solution

*   **Cloud Storage (Recommended):** AWS S3, Google Cloud Storage, or Azure Blob Storage.
    *   **Pros:** Scalability, reliability, security features, direct client upload capabilities (via pre-signed URLs).
    *   The backend will generate pre-signed URLs for uploads, and store the final `photo_url`.
*   **Local Storage (Alternative for development/on-premise):**
    *   Photos stored on the server's filesystem. Simpler to set up initially but complex for scaling and managing in production for web apps.
*   **Decision for this design:** Assume **AWS S3** as a common cloud choice. The `photo_url` will be an S3 URL.

### 3. API Endpoints (Backend - Node.js/Express)

*   **`POST /api/customers/{customerId}/photos/presigned-url` (Get Pre-signed URL for Upload)**
    *   Permissions: Admin, Staff.
    *   Request Body: `{ "fileName": "profile_before.jpg", "fileType": "image/jpeg" }`
    *   Logic:
        1.  Generates a unique key for S3 (e.g., `customer_{customerId}/photos/{uuid}_{fileName}`).
        2.  Uses AWS SDK to generate a pre-signed S3 PUT URL for that key and file type.
        3.  Returns the pre-signed URL and the final `photo_url` (the S3 URL after upload).
    *   Response (200 OK): `{ "uploadUrl": "s3_presigned_put_url", "photoUrl": "final_s3_object_url" }`

*   **`POST /api/customers/{customerId}/photos` (Register Photo Metadata)**
    *   Permissions: Admin, Staff.
    *   Request Body: `{ "photoUrl": "final_s3_object_url_from_previous_step", "description": "...", "serviceId": null, "appointmentId": null, "uploadedByStaffId": X }`
    *   Logic:
        1.  This endpoint is called *after* the client has successfully uploaded the photo to S3 using the pre-signed URL.
        2.  Creates a record in the `CustomerPhotos` table with the provided `photoUrl` and other metadata.
    *   Response (201 Created): The new `CustomerPhotos` record.

*   **`GET /api/customers/{customerId}/photos`**
    *   Permissions: Admin, Staff, Customer (for their own ID).
    *   Response (200 OK): Array of `CustomerPhotos` objects for the customer.

*   **`DELETE /api/photos/{photoId}`**
    *   Permissions: Admin, Staff (potentially with restrictions, e.g., only delete photos they uploaded unless admin).
    *   Logic:
        1.  Fetch `CustomerPhotos` record to get `photo_url`.
        2.  Delete the photo file from S3 using the AWS SDK.
        3.  Delete the record from the `CustomerPhotos` table.
    *   Response (204 No Content) or (404 Not Found).

### 4. UI (Frontend - Admin/Staff React Components)

*   **Customer Detail View (`CustomerDetailPage`):**
    *   Add a "Photo Gallery" or "Documentation" section.
    *   **`PhotoList` Component:** Displays thumbnails of photos fetched from `GET /api/customers/{customerId}/photos`. Clicking a thumbnail could open a larger view in a modal. Each photo shows description, upload date, and linked service/appointment if any.
    *   **`PhotoUploadModal` Component:**
        1.  File input for selecting photo.
        2.  Fields for `description`, optional dropdowns for `serviceId` and `appointmentId`.
        3.  On selecting a file:
            a.  Client calls `POST /api/customers/{customerId}/photos/presigned-url` to get `uploadUrl` and `photoUrl`.
            b.  Client uploads the file directly to S3 using the `uploadUrl`.
            c.  On successful S3 upload, client calls `POST /api/customers/{customerId}/photos` with the `photoUrl` and other metadata.
        4.  Updates the gallery upon successful registration.
    *   Each photo in the list/gallery has a "Delete" button, calling `DELETE /api/photos/{photoId}` after confirmation.

## III. Birthday/Anniversary Reminders (Notifications)

### 1. Data Model (PostgreSQL)

*   **`Users` Table (for `role = 'customer'`):**
    *   Add/Confirm fields:
        *   `birth_date` (DATE, Nullable)
        *   `anniversary_date` (DATE, Nullable): This could represent customer's first visit date or loyalty program start date.
        *   `last_birthday_notification_sent_year` (INTEGER, Nullable): Store the year the last birthday notification was sent.
        *   `last_anniversary_notification_sent_year` (INTEGER, Nullable): Store the year the last anniversary notification was sent.

    *SQL conceptual additions to `Users` table:*
    ```sql
    ALTER TABLE Users
    ADD COLUMN birth_date DATE,
    ADD COLUMN anniversary_date DATE,
    ADD COLUMN last_birthday_notification_sent_year INTEGER,
    ADD COLUMN last_anniversary_notification_sent_year INTEGER;
    ```

### 2. Scheduler/Cron Job (Backend - Node.js)

*   **Technology:** `node-cron`.
*   **Frequency:** Runs daily (e.g., early morning).
*   **Logic (`DailyReminderJob`):**
    1.  **Birthdays:**
        *   Get current date (day and month).
        *   Query `Users` table:
            *   Where `EXTRACT(MONTH FROM birth_date) = currentMonth` AND `EXTRACT(DAY FROM birth_date) = currentDay`.
            *   AND (`last_birthday_notification_sent_year IS NULL` OR `last_birthday_notification_sent_year < currentYear`).
            *   AND `role = 'customer'`.
            *   AND `email_notifications_enabled = true` (or `sms_notifications_enabled = true` if using SMS).
        *   For each matching customer:
            *   Trigger birthday email (using `EmailService`).
            *   (Optional) Trigger SMS (using `SMSService` if integrated).
            *   Update `last_birthday_notification_sent_year` to the current year for that user.
    2.  **Anniversaries:**
        *   Similar logic as birthdays, using `anniversary_date` and `last_anniversary_notification_sent_year`.
    3.  **Error Handling:** Log any errors during the process.
    4.  **Batching/Rate Limiting:** If many notifications, send in batches or consider rate limits of email/SMS providers.

### 3. Notification Tracking

*   The `last_birthday_notification_sent_year` and `last_anniversary_notification_sent_year` fields in the `Users` table prevent sending multiple notifications for the same event in the same year.

### 4. Email/SMS Templates

*   **Location:** Store basic templates in configuration files or a simple database table (e.g., `NotificationTemplates`).
*   **Birthday Template (Conceptual):**
    *   Subject: "Happy Birthday, [Customer Name]! 🎉"
    *   Body: "Wishing you a fantastic birthday from all of us at [Salon Name]! [Optional: Include a special offer like 'Enjoy 10% off your next visit this month.']"
*   **Anniversary Template (Conceptual):**
    *   Subject: "Happy [Anniversary Type] Anniversary, [Customer Name]!"
    *   Body: "Thank you for being a valued customer for [X] years! We appreciate you. [Optional: Special offer.]"
*   Personalization: Templates should allow placeholders for customer name, salon name, offer details, etc.

### 5. UI (Frontend - Admin/Staff React Components)

*   **Customer Profile Edit Form (`CustomerDetailPage`):**
    *   Fields to view/edit `birth_date` and `anniversary_date`.
*   **Dashboard Widget / Upcoming Events Section:**
    *   Display a list of customers with birthdays or anniversaries in the upcoming week/month.
    *   Fetched via a new API endpoint like `GET /api/customers/upcoming-events?type=birthday&days_ahead=7`.
*   **Manual Trigger (Optional):**
    *   On the customer profile or dashboard list, allow staff to manually trigger sending a pre-defined birthday/anniversary message if needed (e.g., if a customer just signed up and their birthday is tomorrow). This would bypass the `last_..._sent_year` check for that manual send but should still update it.

## IV. Deliverables Summary

*   This document (`crm_enhancements_design.md`) outlining:
    *   **Allergy/Sensitivity Tracking:**
        *   `Users` table: `allergies_sensitivities` (JSONB).
        *   API updates to `GET`/`PUT /api/customers/{customerId}`.
        *   UI for viewing/managing allergies and alerts during booking.
    *   **Photo Documentation:**
        *   `CustomerPhotos` table schema.
        *   Discussion of cloud storage (S3) and pre-signed URL flow.
        *   APIs: `POST .../presigned-url`, `POST .../photos` (metadata), `GET .../photos`, `DELETE /api/photos/{photoId}`.
        *   UI for photo gallery, upload, and deletion.
    *   **Birthday/Anniversary Reminders:**
        *   `Users` table: `birth_date`, `anniversary_date`, `last_..._sent_year` fields.
        *   Design for daily cron job (`DailyReminderJob`) using `node-cron`.
        *   Conceptual email/SMS templates.
        *   UI for date input and displaying upcoming events.

This design provides a comprehensive plan for these CRM enhancements.
