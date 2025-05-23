# Service Management Module - API and Frontend Design

This document details the design for managing services, including their catalog, pricing, duration, and basic requirements. It aligns with `project_setup_plan.md`, `appointment_module_design.md`, and `staff_management_module_design.md`.

## I. Data Model Refinements (PostgreSQL)

1.  **`Services` Table:**
    *   **Confirmed and Enhanced Fields:**
        *   `id` (SERIAL PRIMARY KEY)
        *   `name` (TEXT NOT NULL): Name of the service.
        *   `description` (TEXT, Nullable): Detailed description of the service.
        *   `duration_minutes` (INTEGER NOT NULL): Duration of the service in minutes.
        *   `price` (DECIMAL(10, 2) NOT NULL): Price of the service.
        *   `category` (TEXT, Nullable): For grouping services (e.g., "Hair Styling", "Manicures", "Facials").
        *   `requirements` (TEXT, Nullable): Simple notes about service prerequisites or needs (e.g., "Patch test required 24h prior", "Client should bring open-toe shoes").
        *   `is_active` (BOOLEAN DEFAULT TRUE NOT NULL): To soft delete or temporarily hide services from customers.
        *   `created_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP)
        *   `updated_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP)

    *SQL for `Services` Table:*
    ```sql
    CREATE TABLE Services (
        id SERIAL PRIMARY KEY,
        name TEXT NOT NULL,
        description TEXT,
        duration_minutes INTEGER NOT NULL CHECK (duration_minutes > 0),
        price DECIMAL(10, 2) NOT NULL CHECK (price >= 0),
        category TEXT,
        requirements TEXT,
        is_active BOOLEAN DEFAULT TRUE NOT NULL,
        created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
    );

    CREATE INDEX idx_services_category ON Services(category);
    CREATE INDEX idx_services_is_active ON Services(is_active);
    ```

2.  **`StaffServices` Table (from `staff_management_module_design.md`):**
    *   **Reconfirmed:** This table is essential and correctly designed for linking staff members to services they are qualified to perform.
        *   `staff_id` (INTEGER, REFERENCES `Users`(`id`) ON DELETE CASCADE)
        *   `service_id` (INTEGER, REFERENCES `Services`(`id`) ON DELETE CASCADE)
        *   `PRIMARY KEY (staff_id, service_id)`
    *   This linkage is critical for:
        *   Determining which staff can be booked for a particular service (used in `GET /api/availability` in `appointment_module_design.md`).
        *   Displaying services a staff member can perform on their profile (if needed).
        *   Potentially filtering services based on staff availability (advanced).

## II. Backend API Development (Node.js/Express & PostgreSQL)

These APIs are primarily for Admin users to manage the service catalog. Authentication (`ensureAuthenticated`) and Authorization (`ensureRole(['admin'])`) middleware will protect these endpoints, except where noted for public/customer access.

1.  **`POST /api/services` (Create Service)**
    *   Description: Adds a new service to the catalog.
    *   Permissions: Admin only.
    *   Request Body:
        ```json
        {
          "name": "Deluxe Manicure",
          "description": "Includes hand soak, nail shaping, cuticle care, massage, and polish.",
          "duration_minutes": 60,
          "price": 45.00,
          "category": "Nail Care",
          "requirements": "Please remove old polish before appointment.", // Optional
          "is_active": true // Optional, defaults to true
        }
        ```
    *   Response (201 Created):
        ```json
        {
          "id": 123,
          "name": "Deluxe Manicure",
          "description": "Includes hand soak, nail shaping, cuticle care, massage, and polish.",
          "duration_minutes": 60,
          "price": "45.00",
          "category": "Nail Care",
          "requirements": "Please remove old polish before appointment.",
          "is_active": true,
          "created_at": "YYYY-MM-DDTHH:mm:ss.sssZ",
          "updated_at": "YYYY-MM-DDTHH:mm:ss.sssZ"
        }
        ```
    *   Error Responses: 400 Bad Request (validation errors), 401 Unauthorized, 403 Forbidden.

2.  **`GET /api/services` (List Services)**
    *   Description: Retrieves a list of services.
    *   Permissions:
        *   Admin/Staff: Can see all services, including inactive ones, for management purposes.
        *   Customer/Public: Should only see `is_active: true` services.
    *   Query Parameters:
        *   `category` (string): Filter by category name (e.g., `/api/services?category=Hair Styling`).
        *   `is_active` (boolean): Filter by active status (e.g., `/api/services?is_active=true`). Admins can use `is_active=false`. If not provided, public users see active only; admins see all.
        *   `name` (string): Search by service name (partial match).
    *   Response (200 OK):
        ```json
        [
          {
            "id": 123,
            "name": "Deluxe Manicure",
            "description": "...",
            "duration_minutes": 60,
            "price": "45.00",
            "category": "Nail Care",
            "requirements": "...",
            "is_active": true,
            "created_at": "...",
            "updated_at": "..."
          }
          // ... more services
        ]
        ```

3.  **`GET /api/services/{serviceId}` (Get Single Service)**
    *   Description: Retrieves details for a specific service.
    *   Permissions: Public or Customer/Staff/Admin. If the service is `is_active: false`, only Admin/Staff should be able to view it.
    *   Response (200 OK): Single service object (same structure as in `POST` response) or 404 Not Found.

4.  **`PUT /api/services/{serviceId}` (Update Service)**
    *   Description: Updates an existing service.
    *   Permissions: Admin only.
    *   Request Body: Partial or full service object with fields to update.
        ```json
        {
          "price": 50.00,
          "description": "Updated description: Now includes a hydrating mask.",
          "is_active": true
        }
        ```
    *   Response (200 OK): Updated service object or 404 Not Found.

5.  **`DELETE /api/services/{serviceId}` (Deactivate/Delete Service)**
    *   Description: Deactivates a service (soft delete).
    *   Permissions: Admin only.
    *   Strategy: **Soft delete**. The service record will have `is_active` set to `false`. It will not be visible to customers for booking and will not appear in default public listings.
        *   This preserves historical data (e.g., past appointments with this service).
        *   It allows the service to be reactivated later if needed.
        *   Entries in `StaffServices` linking staff to this service can remain. If a service is reactivated, these links are still valid. The booking availability logic must only consider staff linked to *active* services.
    *   Response (200 OK):
        ```json
        {
          "message": "Service deactivated successfully.",
          "service": { /* updated service object with is_active: false */ }
        }
        ```
        or (204 No Content) or (404 Not Found).
    *   Note: A hard delete endpoint is generally not recommended for services due to relationships with appointments and staff. If absolutely necessary, it would require careful handling of foreign key constraints or setting them to `ON DELETE SET NULL` or `ON DELETE CASCADE` where appropriate (e.g., `StaffServices` might use `ON DELETE CASCADE`).

## III. Frontend Outline (React)

### 1. Service Management Components (for Admin View - e.g., in `/admin/services`)

*   **`AdminServiceListPage` Component:**
    *   Route: `/admin/services`
    *   Fetches services using `GET /api/services` (with params to show all, including inactive).
    *   **Filters:** Dropdown for `category`, toggle/checkbox for `is_active` status, search input for `name`.
    *   **Displays `ServiceTable` Component:**
        *   Props: `services` (array from API).
        *   Renders a table with columns: Name, Category, Duration (minutes), Price, Requirements (tooltip/shortened), Active Status.
        *   Each row has "Edit" and "Deactivate/Activate" buttons. "Edit" opens `ServiceFormModal`. "Deactivate/Activate" calls `DELETE /api/services/{serviceId}` (to set `is_active=false`) or `PUT /api/services/{serviceId}` (to set `is_active=true`).
    *   **"Add New Service" Button:** Opens `ServiceFormModal` for creating a new service.

*   **`ServiceFormModal` Component (or dedicated `/admin/services/new`, `/admin/services/{serviceId}/edit` pages):**
    *   Props: `serviceData` (for editing, null for new), `isOpen`, `onClose`, `onSave`.
    *   Form with fields for:
        *   `name` (Text Input)
        *   `description` (Textarea)
        *   `duration_minutes` (Number Input)
        *   `price` (Number Input, formatted for currency)
        *   `category` (Text Input or Dropdown if categories become predefined)
        *   `requirements` (Textarea)
        *   `is_active` (Checkbox)
    *   On submit, calls `POST /api/services` (if new) or `PUT /api/services/{serviceId}` (if editing).
    *   Handles validation, loading states, and error messages.
    *   **Note on Staff Assignment:** Linking services to staff is managed within the Staff Management module (`PUT /api/staff/{staffId}` by providing `assignedServiceIds`), not directly in this service form. This keeps concerns separate. An admin defines a service here, then goes to a staff member's profile to indicate they can perform that service.

### 2. Service Display Components (for Customer/Public View)

*   **`ServiceList` / `ServiceMenu` Component (as mentioned in `appointment_module_design.md`):**
    *   Used in the online booking flow.
    *   Fetches active services using `GET /api/services?is_active=true`.
    *   Can be enhanced with category filtering/grouping.
    *   Displays service name, description, duration, price.
    *   Each service item, when selected, proceeds to the availability checking step in the booking flow.

## IV. Deliverables Summary for this Document (`service_management_module_design.md`)

*   **Detailed `Services` Table Schema:** Confirmed fields and added `category`, `requirements`, and `is_active`.
*   **Reconfirmation of `StaffServices` Table:** Its role in linking staff to services is crucial.
*   **Detailed Service CRUD API Endpoints:**
    *   `POST /api/services`
    *   `GET /api/services` (with filtering)
    *   `GET /api/services/{serviceId}`
    *   `PUT /api/services/{serviceId}`
    *   `DELETE /api/services/{serviceId}` (soft delete strategy by setting `is_active=false`).
    *   Clear request/response payloads and permission considerations for each.
*   **Outline of React Components for Service Management:**
    *   Admin Views: `AdminServiceListPage` (with table and filters) and `ServiceFormModal` (for create/edit).
    *   Customer Views: Referenced existing `ServiceList` component for booking.
*   Clarification that staff-to-service linking is managed via the Staff Management module.

This design provides a comprehensive plan for managing the service catalog within the salon system.
