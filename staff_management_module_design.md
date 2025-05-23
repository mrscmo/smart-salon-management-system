# Staff Management Module - API and Frontend Design

This document outlines the backend APIs and frontend components for Basic Staff Management, focusing on employee profiles, skill sets, and their linkage to services. It complements `appointment_module_design.md` and `project_setup_plan.md`.

## I. Data Model Refinements (PostgreSQL)

1.  **`Users` Table (for Staff role):**
    *   The existing `Users` table (from `project_setup_plan.md`) will be used for core staff information. When a user has `role = 'staff'`, additional staff-specific fields will be relevant.
    *   **Essential fields already in `Users`:**
        *   `id` (Primary Key)
        *   `first_name` (String)
        *   `last_name` (String)
        *   `email` (String, Unique, Indexed)
        *   `phone_number` (String, Optional)
        *   `role` (Enum/String: 'admin', **'staff'**, 'customer')
    *   **New fields to add to `Users` table for staff-specific details:**
        *   `skill_sets` (TEXT[], Nullable): An array of strings representing simple skill tags (e.g., `{"hair cutting", "coloring", "manicure"}`). This is suitable for the "simple tags" requirement.
        *   `bio` (TEXT, Nullable): A short biography or notes about the staff member, visible internally and potentially on a public-facing staff profile page.
        *   `is_active` (BOOLEAN, Default: true): For staff, this can indicate if they are currently active employees. This is separate from `is_verified` for email.

    *Updated `Users` Table structure snippet (conceptual additions):*
    ```sql
    ALTER TABLE Users
    ADD COLUMN skill_sets TEXT[],
    ADD COLUMN bio TEXT,
    ADD COLUMN is_active BOOLEAN DEFAULT true; -- Ensure this applies contextually, e.g. only for staff role
    ```
    *(Note: The `is_active` field might already exist or be planned; if so, its usage for staff should be confirmed. If it's globally for users, it might mean 'account active' vs. 'employee active'. For now, assume it's suitable for marking staff as active/inactive employees).*

2.  **`Services` Table:**
    *   No changes are assumed for the `Services` table itself from `appointment_module_design.md` for this module, but its linkage is critical.

3.  **`StaffServices` Table (Many-to-Many Join Table):**
    *   **Purpose:** To explicitly link which staff members are qualified or assigned to perform which services. This is more robust and queryable than relying solely on matching text tags in `Users.skill_sets` to some property on `Services`. It's crucial for the `GET /api/availability?serviceId=X&date=YYYY-MM-DD` endpoint.
    *   **Schema:**
        *   `staff_id` (INTEGER, REFERENCES `Users`(`id`) ON DELETE CASCADE)
        *   `service_id` (INTEGER, REFERENCES `Services`(`id`) ON DELETE CASCADE)
        *   `PRIMARY KEY (staff_id, service_id)`
        *   `created_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP)

    *SQL for `StaffServices` Table:*
    ```sql
    CREATE TABLE StaffServices (
        staff_id INTEGER NOT NULL REFERENCES Users(id) ON DELETE CASCADE,
        service_id INTEGER NOT NULL REFERENCES Services(id) ON DELETE CASCADE,
        PRIMARY KEY (staff_id, service_id),
        created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
    );
    ```

## II. Backend API Development (Node.js/Express & PostgreSQL)

Authentication (`ensureAuthenticated`) and Authorization (`ensureRole`) middleware will protect these endpoints.

### 1. Staff Profile API Endpoints

*   **`GET /api/staff`**
    *   Description: List all active staff members.
    *   Permissions: Admin, Staff (Staff might see a list of colleagues).
    *   Query Parameters: `include_inactive` (boolean, defaults to false, for Admins).
    *   Response (200 OK):
        ```json
        [
          {
            "id": 1,
            "firstName": "Jane",
            "lastName": "Doe",
            "email": "jane.doe@example.com",
            "phoneNumber": "123-456-7890", // Optional
            "skillSets": ["hair cutting", "coloring"], // From Users.skill_sets
            "isActive": true,
            "assignedServices": [ // Derived from StaffServices table
                { "serviceId": 10, "serviceName": "Women's Haircut" },
                { "serviceId": 12, "serviceName": "Full Color" }
            ]
          }
          // ... more staff
        ]
        ```
    *   Logic: Fetches users where `role = 'staff'`. If `include_inactive` is false, filters by `is_active = true`. Joins with `StaffServices` and `Services` to list assigned services.

*   **`GET /api/staff/{staffId}`**
    *   Description: Get detailed profile for a specific staff member.
    *   Permissions: Admin, Staff (can view their own or other staff profiles depending on policy).
    *   Response (200 OK):
        ```json
        {
          "id": 1,
          "firstName": "Jane",
          "lastName": "Doe",
          "email": "jane.doe@example.com",
          "phoneNumber": "123-456-7890", // Optional
          "skillSets": ["hair cutting", "coloring"],
          "bio": "Experienced stylist with 10 years in the industry.",
          "isActive": true,
          "assignedServices": [
             { "serviceId": 10, "serviceName": "Women's Haircut" },
             { "serviceId": 12, "serviceName": "Full Color" }
          ],
          "createdAt": "YYYY-MM-DDTHH:mm:ss.sssZ",
          "updatedAt": "YYYY-MM-DDTHH:mm:ss.sssZ"
        }
        ```
        or (404 Not Found if `staffId` is not a staff member).
    *   Logic: Fetches user by `id` where `role = 'staff'`. Joins with `StaffServices` and `Services`.

*   **`PUT /api/staff/{staffId}`**
    *   Description: Update staff profile information.
    *   Permissions: Admin (can edit any staff), Staff (can edit their own profile - `staffId` must match authenticated user's ID).
    *   Request Body:
        ```json
        {
          "firstName": "Jane", // Admins might update this
          "lastName": "Doe",   // Admins might update this
          "phoneNumber": "123-456-0000",
          "skillSets": ["hair cutting", "coloring", "highlights"], // Replaces existing skills
          "bio": "Specializes in modern color techniques.",
          "isActive": true, // Admin only
          "assignedServiceIds": [10, 12, 15] // Array of Service IDs. Replaces existing assignments in StaffServices.
        }
        ```
    *   Response (200 OK): Updated staff object (as in `GET /api/staff/{staffId}`).
    *   Logic:
        1.  Verify permissions.
        2.  Update relevant fields in the `Users` table for the given `staffId`.
        3.  If `assignedServiceIds` is provided:
            *   Delete existing entries for this `staffId` from `StaffServices`.
            *   Insert new entries into `StaffServices` for each ID in `assignedServiceIds`.
        4.  Fetch and return the updated staff profile.

*   **Staff Creation:**
    *   As stated in the prompt, creating a user with the 'staff' role is typically an Admin function via a general user management interface. This might involve:
        1.  Admin creates a new user (e.g., `POST /api/users`) or updates an existing user (`PUT /api/users/{userId}`).
        2.  Sets the `role` to 'staff'.
        3.  The new staff member (or Admin) can then use `PUT /api/staff/{staffId}` to populate staff-specific details like `bio`, `skillSets`, and `assignedServiceIds`.
    *   No dedicated `POST /api/staff` for creation is planned to avoid redundancy with general user creation.

### 2. Skills Management API Endpoints

*   Given the decision to use `skill_sets` (TEXT[]) as "simple tags" on the `Users` table, dedicated API endpoints for managing a separate `Skills` table are **not required** for this iteration.
*   Skills are managed directly via the `skillSets` array in the `PUT /api/staff/{staffId}` endpoint.
*   If, in the future, skills require descriptions, categories, or more complex management, a `Skills` table and corresponding APIs (`GET /api/skills`, `POST /api/skills`, etc.) would be introduced. At that point, `Users.skill_sets` might become a list of foreign keys (Skill IDs).

## III. Frontend Outline (React)

### 1. Staff Profile Management Components (for Admin View - typically in `/admin/staff`)

*   **`StaffListPage` Component:**
    *   Route: `/admin/staff`
    *   Fetches staff using `GET /api/staff`. Includes option to show inactive staff.
    *   Displays `StaffTable` component.
    *   May include a button/link to "Add New Staff" (which might navigate to a general user creation form with 'staff' role pre-selected, or a specific staff onboarding flow).
*   **`StaffTable` Component:**
    *   Props: `staffMembers` (array from API).
    *   Renders a table: Name, Email, Phone, Skill Sets (comma-separated or tags), Active Status.
    *   Each row links to `StaffProfileEditPage` for that staff member.
*   **`StaffProfileEditPage` Component:**
    *   Route: `/admin/staff/{staffId}/edit`
    *   Fetches staff details using `GET /api/staff/{staffId}`.
    *   Provides a form with fields for:
        *   First Name, Last Name, Email (possibly read-only or carefully managed).
        *   Phone Number.
        *   Skill Sets (e.g., a tag input component).
        *   Bio (textarea).
        *   Is Active (checkbox, for Admins).
        *   Assigned Services (e.g., a multi-select dropdown or checklist fetched from `GET /api/services`).
    *   On submit, calls `PUT /api/staff/{staffId}` with the updated data. Handles success and error messages.

### 2. Staff-Facing Profile Components (for logged-in Staff - typically in `/profile` or `/my-staff-profile`)

*   **`MyStaffProfilePage` Component:**
    *   Route: `/my-profile` (if combined with general user profile) or a staff-specific route.
    *   Fetches their own data using `GET /api/staff/{auth.userId}`.
    *   Displays:
        *   Non-editable info: Name, Email (as per policy).
        *   Editable info: Phone Number, Bio.
        *   Skill Sets: Displayed. Editable if business rules allow (e.g., a tag input).
        *   Assigned Services: Displayed (usually not editable by staff directly, but by Admin).
    *   Form for editing uses `PUT /api/staff/{auth.userId}`.
    *   Links to their schedule/availability management (already designed in `appointment_module_design.md`).

## IV. Clarification on Skills and Service Linkage

*   **Primary Linkage:** The **`StaffServices` table** provides the explicit, primary link between a staff member and the services they are qualified/assigned to perform.
    *   `StaffServices.staff_id` (FK to `Users.id`)
    *   `StaffServices.service_id` (FK to `Services.id`)
*   **Role of `Users.skill_sets` (TEXT[]):**
    *   These are descriptive tags that can be used for display on staff profiles, internal searching/filtering by admins, or as a quick reference.
    *   They are **not** the primary mechanism for determining service capability for booking. For example, a service "Advanced Chemical Peel" might require specific certifications that are implicitly covered by its assignment in `StaffServices`, while `skill_sets` might just list "skincare" or "peels".
*   **Appointment Booking Logic (`GET /api/availability?serviceId=X&date=YYYY-MM-DD`):**
    1.  When this endpoint is called with a `serviceId`:
    2.  The system will query the `StaffServices` table to find all `staff_id`s associated with the given `serviceId`.
    3.  For each of these qualified staff members, their availability (recurring, overrides, existing appointments) will then be checked as detailed in `appointment_module_design.md`.
    4.  This ensures that only staff who are explicitly assigned to perform a service are considered for availability for that service.

## V. Deliverables Summary for this Document (`staff_management_module_design.md`)

*   **Database Schema Updates:**
    *   `Users` table: Added `skill_sets` (TEXT[]), `bio` (TEXT), `is_active` (BOOLEAN).
    *   New `StaffServices` table: `staff_id` (FK), `service_id` (FK) to link staff to services.
*   **Defined API Endpoints for Staff Management:**
    *   `GET /api/staff`
    *   `GET /api/staff/{staffId}`
    *   `PUT /api/staff/{staffId}`
    *   Request/response structures and permissions outlined.
    *   Clarification on staff creation (via general user management).
*   **No dedicated Skills API needed** for this iteration due to "simple tags" approach.
*   **Outline of React Components for Staff Profile Management:**
    *   Admin Views: `StaffListPage`, `StaffTable`, `StaffProfileEditPage`.
    *   Staff Views: `MyStaffProfilePage`.
*   **Clarification on Skills and Service Linkage:**
    *   `StaffServices` table is the primary link for service capability.
    *   `Users.skill_sets` are descriptive tags.
    *   Appointment booking logic relies on `StaffServices`.

This design provides a plan for managing staff profiles and their association with services, integrating with the overall system architecture.
