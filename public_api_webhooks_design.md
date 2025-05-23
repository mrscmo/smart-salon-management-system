# Public/Partner API Access and Webhook System - Design Document

This document outlines the design for a public/partner-facing API and a corresponding webhook system to allow external integrations.

## I. Public/Partner API Design

### 1. API Versioning Strategy

*   **Strategy:** URL Path Versioning.
*   **Implementation:** All partner API endpoints will be prefixed with `/api/v1/`. For example: `/api/v1/partner/services`.
*   **Initial Version:** `v1`. Future breaking changes will increment the version (e.g., `/api/v2/...`). Non-breaking changes can be added to the current version.

### 2. Authentication & Authorization for External Partners

*   **Mechanism:** API Keys.
*   **`PartnerAccounts` Table (PostgreSQL):**
    *   Stores information about approved external partners and their API access credentials.
    *   **Schema:**
        *   `id` (SERIAL PRIMARY KEY)
        *   `partner_name` (TEXT NOT NULL UNIQUE, e.g., "Hotel Concierge Inc.", "Affiliate Booker Ltd.")
        *   `api_key_hash` (TEXT NOT NULL UNIQUE): Hashed version of the generated API key (e.g., using bcrypt or SHA256). The raw key is shown to the partner once upon generation.
        *   `api_key_prefix` (TEXT NOT NULL UNIQUE): A short, unique, non-secret prefix for the API key (e.g., `partner_pk_`) to help identify the key in logs or if a partner leaks it.
        *   `is_active` (BOOLEAN DEFAULT TRUE NOT NULL)
        *   `permissions` (JSONB NOT NULL DEFAULT '{}'::jsonb): Stores granted permissions.
            *   Example: `{"appointments": ["read_availability", "create_booking", "read_own_booking"], "services": ["read_list"]}`
            *   Or simpler array: `["read:appointments_availability", "write:appointments_booking", "read:appointments_own_booking", "read:services_list"]`
        *   `contact_email` (TEXT, Nullable)
        *   `created_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)
        *   `updated_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)

    *SQL for `PartnerAccounts` Table:*
    ```sql
    CREATE TABLE PartnerAccounts (
        id SERIAL PRIMARY KEY,
        partner_name TEXT NOT NULL UNIQUE,
        api_key_hash TEXT NOT NULL UNIQUE,
        api_key_prefix TEXT NOT NULL UNIQUE, -- e.g., 8-10 chars
        is_active BOOLEAN DEFAULT TRUE NOT NULL,
        permissions JSONB NOT NULL DEFAULT '{}'::jsonb,
        contact_email TEXT,
        created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
        updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL
    );
    CREATE INDEX idx_partneraccounts_is_active ON PartnerAccounts(is_active);
    ```

*   **API Key Handling:**
    1.  **Generation:** Admin generates an API key for a partner. The key is displayed once to the partner and should be stored securely by them. The system stores only the `api_key_hash` and `api_key_prefix`.
    2.  **Transmission:** Partners must include the API key in their requests via the `Authorization` header: `Authorization: Bearer <api_key>`. (Alternatively, `X-API-KEY: <api_key>` could be supported).
*   **Authentication Middleware (Backend):**
    1.  Extract the API key from the request header.
    2.  (Optional, for efficiency) Use the prefix to quickly look up the partner if there are many keys.
    3.  Compare the provided key with the stored `api_key_hash` for all active partners (or the specific partner found by prefix). Use a constant-time comparison function if hashing directly.
    4.  If a match is found and `is_active` is true, the partner is authenticated. Attach partner details (ID, permissions) to the request object.
    5.  If no match or partner is inactive, return 401 Unauthorized.
*   **Authorization Middleware (Backend):**
    1.  After authentication, check if the authenticated partner's `permissions` allow the requested action on the resource (e.g., permission "write:appointments_booking" for `POST /api/v1/partner/bookings`).
    2.  If not permitted, return 403 Forbidden.

### 3. Key Endpoints to Expose (Version `v1`)

These endpoints are designed for specific, limited integration scenarios, not full salon management.

*   **Appointments:**
    *   **`POST /api/v1/partner/bookings`**
        *   Purpose: Allow a partner to create a booking.
        *   Request Body (Constrained):
            ```json
            {
              "serviceId": 123,
              "staffId": 45, // Optional, if partner can specify or system assigns
              "startTime": "YYYY-MM-DDTHH:mm:ssZ", // Specific start time from availability
              "customer": { // Partner provides customer details
                "firstName": "John",
                "lastName": "Doe",
                "email": "john.doe.partner@example.com", // Partner might use a unique email or their own
                "phoneNumber": "123-456-7890"
              },
              "partnerBookingReference": "hotel_ref_abc" // Optional reference from partner system
            }
            ```
        *   Logic:
            1.  Backend validates availability for `serviceId`, `staffId`, `startTime`.
            2.  Finds or creates a customer record based on provided details. If creating, `Users.role` is 'customer'. Associate customer with partner if needed for tracking.
            3.  Creates an appointment, potentially flagging it as `booked_by_partner_id = authenticatedPartner.id`.
        *   Response (201 Created): Appointment details, including salon's `bookingId`.
    *   **`GET /api/v1/partner/availability`**
        *   Purpose: Expose service availability.
        *   Query Parameters: `serviceId` (required), `date` (required, YYYY-MM-DD), `staffId` (optional).
        *   Response (200 OK): Similar to internal `GET /api/availability` (from `appointment_module_design.md`), returning list of available slots:
            ```json
            {
              "date": "YYYY-MM-DD",
              "serviceId": 123,
              "availableSlots": [
                { "staffId": Y, "staffName": "Jane Doe", "startTime": "09:00", "endTime": "09:30" }
              ]
            }
            ```
    *   **`GET /api/v1/partner/bookings/{bookingId}`**
        *   Purpose: View details of a specific booking made by this partner.
        *   Logic: Ensure the `bookingId` was created by the authenticated partner.
        *   Response (200 OK): Appointment details (salon's view, may differ from customer's view).
    *   **`DELETE /api/v1/partner/bookings/{bookingId}` (Cancel Booking)**
        *   Purpose: Allow partner to cancel a booking they made.
        *   Logic: Check ownership and cancellation policies.
        *   Response (204 No Content) or error.

*   **Services:**
    *   **`GET /api/v1/partner/services`**
        *   Purpose: List available services (can be a curated list for partners, e.g., only `is_active=true` and `is_externally_bookable=true` - new field in `Services`).
        *   Response (200 OK): Array of service objects (ID, name, description, duration, price).

*   **Customers (Very Limited):**
    *   The `POST /api/v1/partner/bookings` endpoint already handles implicit customer creation/matching. No separate `POST /api/v1/partner/customers` might be needed for MVP to keep customer data handling tight. If absolutely required, it must have very limited fields and clear consent mechanisms. General access to customer lists is disallowed.

### 4. Rate Limiting

*   Implement rate limiting on all `/api/v1/partner/*` endpoints.
*   Strategy: Per API key.
*   Technology: e.g., `express-rate-limit` with a store (like Redis) for distributed environments.
*   Limits: Define reasonable limits (e.g., 100 requests/minute per API key, burstable).

### 5. API Documentation

*   **Strategy:** Use Swagger/OpenAPI specification.
*   **Generation:** Can be auto-generated from code annotations (e.g., using `swagger-jsdoc` and `swagger-ui-express` for Node.js/Express) or manually written.
*   **Content:** Must detail authentication, endpoints, request/response schemas, error codes, and rate limits.
*   **Accessibility:** Host documentation at a public URL for partners.

## II. Webhook System Design

### 1. Webhook Events - Key Events (`v1`)

*   `appointment.created`: Triggered when a new appointment is successfully created (by any source, including partner API).
*   `appointment.updated`: Triggered on significant updates (reschedule, status change like 'confirmed', 'completed').
*   `appointment.cancelled`: Triggered when an appointment is cancelled.
*   `customer.created`: Triggered when a new customer profile is created (especially if by partner).
*   `availability.changed` (Considered Advanced/Future): This can be very noisy. For MVP, partners would typically poll `GET /api/v1/partner/availability`. If implemented, it would need careful scoping (e.g., notify only if a previously unavailable day/slot for a specific service becomes available).

### 2. Data Model (PostgreSQL)

*   **`WebhookSubscriptions` Table:**
    *   **Schema:**
        *   `id` (SERIAL PRIMARY KEY)
        *   `partner_id` (INTEGER NOT NULL REFERENCES `PartnerAccounts`(`id`) ON DELETE CASCADE)
        *   `target_url` (TEXT NOT NULL)
        *   `subscribed_event_types` (TEXT[] NOT NULL CHECK (array_length(subscribed_event_types, 1) > 0)) -- e.g., `{"appointment.created", "appointment.cancelled"}`
        *   `secret` (TEXT NOT NULL DEFAULT encode(gen_random_bytes(32), 'hex')): Generated by system, used for signing payloads.
        *   `is_active` (BOOLEAN DEFAULT TRUE NOT NULL)
        *   `created_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)
        *   `updated_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)

    *SQL for `WebhookSubscriptions` Table:*
    ```sql
    CREATE TABLE WebhookSubscriptions (
        id SERIAL PRIMARY KEY,
        partner_id INTEGER NOT NULL REFERENCES PartnerAccounts(id) ON DELETE CASCADE,
        target_url TEXT NOT NULL,
        subscribed_event_types TEXT[] NOT NULL CHECK (array_length(subscribed_event_types, 1) > 0),
        secret TEXT NOT NULL DEFAULT encode(gen_random_bytes(32), 'hex'),
        is_active BOOLEAN DEFAULT TRUE NOT NULL,
        created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
        updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
        CONSTRAINT fk_webhook_partner FOREIGN KEY (partner_id) REFERENCES PartnerAccounts(id) ON DELETE CASCADE
    );
    CREATE INDEX idx_webhooksubscriptions_partner_id ON WebhookSubscriptions(partner_id);
    CREATE INDEX idx_webhooksubscriptions_is_active ON WebhookSubscriptions(is_active);
    ```

*   **`WebhookDeliveryLogs` Table (New - for tracking attempts):**
    *   **Schema:**
        *   `id` (SERIAL PRIMARY KEY)
        *   `subscription_id` (INTEGER NOT NULL REFERENCES `WebhookSubscriptions`(`id`) ON DELETE CASCADE)
        *   `event_type` (TEXT NOT NULL)
        *   `payload_json` (JSONB NOT NULL)
        *   `target_url` (TEXT NOT NULL)
        *   `attempted_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)
        *   `response_status_code` (INTEGER, Nullable)
        *   `response_body` (TEXT, Nullable)
        *   `status` (TEXT NOT NULL, CHECK (`status` IN ('pending', 'success', 'failed_retryable', 'failed_permanent')))
        *   `retry_count` (INTEGER DEFAULT 0)

    *SQL for `WebhookDeliveryLogs` Table:*
    ```sql
    CREATE TABLE WebhookDeliveryLogs (
        id SERIAL PRIMARY KEY,
        subscription_id INTEGER NOT NULL REFERENCES WebhookSubscriptions(id) ON DELETE CASCADE,
        event_type TEXT NOT NULL,
        payload_json JSONB NOT NULL,
        target_url TEXT NOT NULL,
        attempted_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
        response_status_code INTEGER,
        response_body TEXT,
        status TEXT NOT NULL CHECK (status IN ('pending', 'success', 'failed_retryable', 'failed_permanent')),
        retry_count INTEGER DEFAULT 0,
        CONSTRAINT fk_log_subscription FOREIGN KEY (subscription_id) REFERENCES WebhookSubscriptions(id) ON DELETE CASCADE
    );
    CREATE INDEX idx_webhookdeliverylogs_status_retry ON WebhookDeliveryLogs(status, retry_count);
    ```

### 3. Webhook Delivery Logic (Backend - `WebhookService`)

*   **Event Trigger:** When a defined event occurs (e.g., appointment created), the relevant service (e.g., `AppointmentService`) publishes an internal event or calls `WebhookService`.
*   **Processing:**
    1.  `WebhookService` receives the event (e.g., `event_type = "appointment.created"`, `data = { appointmentObject }`).
    2.  Queries `WebhookSubscriptions` for all active subscriptions for that `event_type`.
    3.  For each subscription:
        *   **Construct Payload:**
            ```json
            // Example for "appointment.created"
            {
              "eventId": "evt_random_uuid_here", // Unique ID for this specific event delivery
              "eventType": "appointment.created",
              "apiVersion": "v1", // Version of your webhook payload structure
              "timestamp": "YYYY-MM-DDTHH:mm:ss.sssZ",
              "data": {
                "bookingId": 12345,
                "serviceId": 101,
                "serviceName": "Deluxe Manicure",
                "staffId": 45,
                "staffName": "Anna K.",
                "startTime": "YYYY-MM-DDTHH:mm:ssZ",
                "endTime": "YYYY-MM-DDTHH:mm:ssZ",
                "customer": { // Key, non-sensitive customer details
                  "partnerProvidedId": "hotel_ref_abc", // If applicable
                  "emailHint": "j...e@example.com" // Masked or hinted if sensitive
                },
                "status": "Confirmed", // Current status
                "bookedByPartnerId": 7 // ID of the partner who made the booking, if applicable
              },
              "salonId": "salon_xyz_main_branch" // Identifier for the salon
            }
            ```
        *   **Sign Payload:** Generate HMAC-SHA256 signature of the JSON payload string using the `WebhookSubscriptions.secret`.
        *   **Asynchronous Delivery:** Use a message queue (e.g., RabbitMQ, Redis Streams, AWS SQS) for robustness. The queue worker will handle HTTP POST requests.
            *   HTTP POST to `target_url`.
            *   Headers: `Content-Type: application/json`, `X-Webhook-Signature: <generated_signature>`.
        *   **Logging:** Create initial `WebhookDeliveryLogs` entry with status 'pending'.
*   **Retry Mechanism (Queue Worker):**
    *   If HTTP POST fails (e.g., timeout, 5xx error from receiver, non-2xx status):
        *   Update `WebhookDeliveryLogs` status to 'failed_retryable', increment `retry_count`.
        *   Retry with exponential backoff (e.g., 1 min, 5 min, 15 min, 1 hr ... up to a max retry count).
    *   If max retries reached or a non-retryable error (e.g., 4xx from receiver indicating bad URL or auth), update status to 'failed_permanent'.
    *   On success (2xx from receiver), update status to 'success'.

### 4. API Endpoints (Backend - for Partners to manage their subscriptions)

All endpoints under `/api/v1/partner/webhooks/subscriptions` and require Partner API Key authentication.

*   **`POST /api/v1/partner/webhooks/subscriptions`**
    *   Request: `{ "target_url": "https://partner.com/webhook-receiver", "event_types": ["appointment.created", "appointment.cancelled"] }`
    *   Logic: Creates a new `WebhookSubscriptions` record for the authenticated `partner_id`. Generates `secret`.
    *   Response (201 Created): `{ "id": X, "target_url": "...", "event_types": [...], "secret": "whsec_generated_secret_here_shown_once", "is_active": true, "created_at": "..." }` (Secret shown only once).
*   **`GET /api/v1/partner/webhooks/subscriptions`**
    *   Logic: Lists subscriptions for the authenticated `partner_id`.
    *   Response (200 OK): Array of subscription objects (secret should NOT be returned here).
*   **`GET /api/v1/partner/webhooks/subscriptions/{subscriptionId}`**
    *   Logic: Get details for a specific subscription owned by the partner.
    *   Response (200 OK): Single subscription object (no secret).
*   **`PUT /api/v1/partner/webhooks/subscriptions/{subscriptionId}`**
    *   Request: `{ "target_url": "...", "event_types": [...], "is_active": true/false }` (Cannot update secret).
    *   Logic: Updates the specified subscription owned by the partner.
    *   Response (200 OK): Updated subscription object (no secret).
*   **`DELETE /api/v1/partner/webhooks/subscriptions/{subscriptionId}`**
    *   Logic: Deletes the subscription owned by the partner.
    *   Response (204 No Content).

## III. Deliverables Summary

*   **API Versioning Strategy:** URL Path versioning, starting with `/api/v1/`.
*   **Partner Authentication:** API Keys (hashed in DB), `PartnerAccounts` table schema, middleware logic.
*   **Database Schemas:** `PartnerAccounts`, `WebhookSubscriptions`, `WebhookDeliveryLogs`.
*   **Public/Partner API Endpoints:**
    *   Appointments: `POST /bookings`, `GET /availability`, `GET /bookings/{id}`, `DELETE /bookings/{id}`.
    *   Services: `GET /services`.
    *   (Customer creation handled implicitly via booking for MVP).
*   **Webhook System:**
    *   Defined Events: `appointment.created`, `appointment.updated`, `appointment.cancelled`, `customer.created`.
    *   Example Payload for `appointment.created`.
    *   Delivery Logic: Event trigger, query subscriptions, construct payload, sign, async delivery via queue, retry mechanism, logging.
    *   Subscription Management APIs: CRUD endpoints for partners to manage their webhook subscriptions (`POST`, `GET` list, `GET` single, `PUT`, `DELETE`).
*   **Plan for API Documentation:** Use Swagger/OpenAPI.
*   **Rate Limiting:** Mentioned for partner APIs.

This design provides a comprehensive plan for Public/Partner API Access and the Webhook system.
