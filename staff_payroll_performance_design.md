# Staff Management Enhancements: Payroll & Performance - Design Document

This document outlines the design for enhancements to Staff Management, focusing on commission structures, calculation, basic time tracking/attendance, and performance metrics. It references `staff_management_module_design.md`, `pos_module_design.md`, and `appointment_module_design.md`.

## I. Data Model Design (PostgreSQL)

### 1. `StaffCommissionRules` Table

*   Stores rules for calculating staff commissions on services and products.
*   **Schema:**
    *   `id` (SERIAL PRIMARY KEY)
    *   `staff_id` (INTEGER, REFERENCES `Users`(`id`) ON DELETE CASCADE, Nullable): If null, rule may apply to all staff or by role (role-based logic applied in service layer).
    *   `service_id` (INTEGER, REFERENCES `Services`(`id`) ON DELETE CASCADE, Nullable): For service-specific commission.
    *   `product_id` (INTEGER, REFERENCES `Products`(`id`) ON DELETE CASCADE, Nullable): For product-specific commission.
    *   `commission_type` (TEXT NOT NULL, CHECK (`commission_type` IN ('percentage', 'fixed_amount'))): E.g., 'percentage' of item price, or 'fixed_amount' per item/service.
    *   `commission_value` (DECIMAL(10, 2) NOT NULL CHECK (`commission_value` >= 0)): If 'percentage', stores value like 10.00 for 10%. If 'fixed_amount', stores the monetary value.
    *   `start_date` (DATE NOT NULL DEFAULT CURRENT_DATE)
    *   `end_date` (DATE, Nullable): If null, the rule does not expire.
    *   `description` (TEXT, Nullable): E.g., "Standard service commission", "Promo product commission".
    *   `is_active` (BOOLEAN DEFAULT TRUE NOT NULL)
    *   `created_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)
    *   `updated_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)
    *   UNIQUE (`staff_id`, `service_id`, `product_id`, `start_date`): To avoid duplicate rules for the same criteria active at the same time. More complex uniqueness might be needed based on overlapping date ranges and nulls. For MVP, this provides a basic constraint.

*SQL for `StaffCommissionRules` Table:*
```sql
CREATE TABLE StaffCommissionRules (
    id SERIAL PRIMARY KEY,
    staff_id INTEGER REFERENCES Users(id) ON DELETE CASCADE,
    service_id INTEGER REFERENCES Services(id) ON DELETE CASCADE,
    product_id INTEGER REFERENCES Products(id) ON DELETE CASCADE,
    commission_type TEXT NOT NULL CHECK (commission_type IN ('percentage', 'fixed_amount')),
    commission_value DECIMAL(10, 2) NOT NULL CHECK (commission_value >= 0),
    start_date DATE NOT NULL DEFAULT CURRENT_DATE,
    end_date DATE,
    description TEXT,
    is_active BOOLEAN DEFAULT TRUE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
    CONSTRAINT check_end_date CHECK (end_date IS NULL OR end_date >= start_date)
    -- Add more specific unique constraints or handle rule precedence in application logic.
    -- Example: UNIQUE (staff_id, service_id, start_date) WHERE product_id IS NULL
);
CREATE INDEX idx_commission_rules_staff ON StaffCommissionRules(staff_id);
CREATE INDEX idx_commission_rules_service ON StaffCommissionRules(service_id);
CREATE INDEX idx_commission_rules_product ON StaffCommissionRules(product_id);
CREATE INDEX idx_commission_rules_active_dates ON StaffCommissionRules(is_active, start_date, end_date);
```

### 2. `StaffCommissionsLedger` Table

*   Records earned commissions for staff.
*   **Schema:**
    *   `id` (SERIAL PRIMARY KEY)
    *   `staff_id` (INTEGER NOT NULL REFERENCES `Users`(`id`) ON DELETE CASCADE)
    *   `transaction_item_id` (INTEGER, REFERENCES `TransactionItems`(`id`) ON DELETE SET NULL, Nullable): Links to the specific POS item.
    *   `appointment_id` (INTEGER, REFERENCES `Appointments`(`id`) ON DELETE SET NULL, Nullable): Links to appointment if commission is for a service not directly in `TransactionItems` or for context.
    *   `commission_rule_id` (INTEGER NOT NULL REFERENCES `StaffCommissionRules`(`id`) ON DELETE RESTRICT)
    *   `base_amount_for_commission` (DECIMAL(10, 2) NOT NULL): The price of the service/product *after item-level discounts* but before transaction-level discounts or tax.
    *   `calculated_commission_amount` (DECIMAL(10, 2) NOT NULL)
    *   `earned_at` (TIMESTAMP WITH TIME ZONE NOT NULL): Timestamp of the completed transaction/appointment.
    *   `payroll_run_id` (INTEGER, Nullable): FK to a future `PayrollRuns` table (out of scope for this design, but placeholder).
    *   `status` (TEXT NOT NULL DEFAULT 'pending' CHECK (`status` IN ('pending', 'approved', 'paid', 'voided')))
    *   `notes` (TEXT, Nullable)
    *   `created_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)

*SQL for `StaffCommissionsLedger` Table:*
```sql
CREATE TABLE StaffCommissionsLedger (
    id SERIAL PRIMARY KEY,
    staff_id INTEGER NOT NULL REFERENCES Users(id) ON DELETE CASCADE,
    transaction_item_id INTEGER REFERENCES TransactionItems(id) ON DELETE SET NULL,
    appointment_id INTEGER REFERENCES Appointments(id) ON DELETE SET NULL,
    commission_rule_id INTEGER NOT NULL REFERENCES StaffCommissionRules(id) ON DELETE RESTRICT,
    base_amount_for_commission DECIMAL(10, 2) NOT NULL,
    calculated_commission_amount DECIMAL(10, 2) NOT NULL,
    earned_at TIMESTAMP WITH TIME ZONE NOT NULL,
    payroll_run_id INTEGER, -- Placeholder for future PayrollRuns table FK
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'approved', 'paid', 'voided')),
    notes TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL
);
CREATE INDEX idx_comm_ledger_staff ON StaffCommissionsLedger(staff_id, status);
CREATE INDEX idx_comm_ledger_earned_at ON StaffCommissionsLedger(earned_at);
CREATE INDEX idx_comm_ledger_transaction_item ON StaffCommissionsLedger(transaction_item_id);
```

### 3. `StaffTimeClockEntries` Table

*   Records staff clock-in and clock-out times.
*   **Schema:**
    *   `id` (SERIAL PRIMARY KEY)
    *   `staff_id` (INTEGER NOT NULL REFERENCES `Users`(`id`) ON DELETE CASCADE)
    *   `clock_in_time` (TIMESTAMP WITH TIME ZONE NOT NULL)
    *   `clock_out_time` (TIMESTAMP WITH TIME ZONE, Nullable)
    *   `notes_clock_in` (TEXT, Nullable)
    *   `notes_clock_out` (TEXT, Nullable)
    *   `duration_minutes` (INTEGER, Nullable): Calculated upon clock-out.
    *   `is_manual_entry` (BOOLEAN DEFAULT FALSE NOT NULL)
    *   `manual_entry_approved_by_user_id` (INTEGER, REFERENCES `Users`(`id`) ON DELETE SET NULL, Nullable): User who approved the manual entry.
    *   `created_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)
    *   `updated_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)

*SQL for `StaffTimeClockEntries` Table:*
```sql
CREATE TABLE StaffTimeClockEntries (
    id SERIAL PRIMARY KEY,
    staff_id INTEGER NOT NULL REFERENCES Users(id) ON DELETE CASCADE,
    clock_in_time TIMESTAMP WITH TIME ZONE NOT NULL,
    clock_out_time TIMESTAMP WITH TIME ZONE,
    notes_clock_in TEXT,
    notes_clock_out TEXT,
    duration_minutes INTEGER,
    is_manual_entry BOOLEAN DEFAULT FALSE NOT NULL,
    manual_entry_approved_by_user_id INTEGER REFERENCES Users(id) ON DELETE SET NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
    CONSTRAINT check_clock_out_after_clock_in CHECK (clock_out_time IS NULL OR clock_out_time > clock_in_time)
);
CREATE INDEX idx_timeclock_staff_time ON StaffTimeClockEntries(staff_id, clock_in_time);
```

## II. Commission Structure & Calculation

### 1. Logic (Backend - `CommissionService`)

*   **Trigger Points:**
    *   When a `Transaction` has `payment_status` updated to 'succeeded' or 'paid_multiple'.
    *   When an `Appointment` (if not linked to a POS transaction item, e.g., a free consultation that still earns commission) is marked 'Completed'.
*   **Calculation Process:**
    1.  For each relevant `TransactionItem` or `Appointment`:
        *   Identify `staff_id` (from `TransactionItems.staff_id` or `Appointments.staff_id`).
        *   Identify `service_id` or `product_id`.
        *   Determine `base_amount_for_commission`: This is `TransactionItems.price_after_discount` for products/services sold via POS. For appointments not in POS, it's the service price.
    2.  **Find Applicable Rule (`StaffCommissionRules`):**
        *   Query rules where `is_active = true` and `earned_at` falls between `start_date` and `end_date` (or `end_date` is null).
        *   **Hierarchy/Precedence (Example):**
            1.  Rule specific to `staff_id`, `service_id`/`product_id`.
            2.  Rule specific to `staff_id`, for ANY service/product (service_id and product_id are NULL).
            3.  Rule for ANY staff (staff_id is NULL), specific to `service_id`/`product_id`.
            4.  Rule for ANY staff, for ANY service/product (all three are NULL - general rule).
            *   This precedence needs to be implemented in the query or application logic. Select the most specific matching rule.
    3.  **Calculate Commission:**
        *   If `commission_type` is 'percentage': `calculated_commission_amount = base_amount_for_commission * (commission_value / 100.0)`.
        *   If `commission_type` is 'fixed_amount': `calculated_commission_amount = commission_value`.
    4.  Create a record in `StaffCommissionsLedger` with status 'pending'.

### 2. API Endpoints (Backend - Admin)

*   **`POST /api/staff-commission-rules`**
    *   Request: `{ staff_id, service_id, product_id, commission_type, commission_value, start_date, end_date, description, is_active }`
    *   Response (201 Created): New rule object.
*   **`GET /api/staff-commission-rules`**
    *   Query Params: `staffId`, `serviceId`, `productId`, `isActive`.
    *   Response (200 OK): Array of rule objects.
*   **`GET /api/staff-commission-rules/{ruleId}`**
    *   Response (200 OK): Single rule object.
*   **`PUT /api/staff-commission-rules/{ruleId}`**
    *   Request: Fields to update.
    *   Response (200 OK): Updated rule object.
*   **`DELETE /api/staff-commission-rules/{ruleId}`**
    *   Logic: Soft delete by setting `is_active = false` is preferred.
    *   Response (204 No Content or 200 OK with updated object).
*   **`GET /api/staff/{staffId}/commissions-ledger`**
    *   Permissions: Admin, or Staff for their own ID.
    *   Query Params: `startDate`, `endDate`, `status`.
    *   Response (200 OK): Array of `StaffCommissionsLedger` entries.
*   **`PUT /api/commissions-ledger/{ledgerEntryId}/status`**
    *   Permissions: Admin.
    *   Request: `{ "status": "approved" | "paid" | "voided" }`
    *   Response (200 OK): Updated ledger entry.
*   **`GET /api/reports/commissions`**
    *   Permissions: Admin.
    *   Query Params: `startDate`, `endDate`, `staffId` (optional), `status` (optional).
    *   Response (200 OK): Aggregated commission data (e.g., total pending, total paid per staff).

## III. Basic Time Tracking & Attendance

### 1. API Endpoints (Backend)

*   **`POST /api/staff/timeclock/clock-in`**
    *   Permissions: Authenticated Staff (for themselves).
    *   Request: `{ notes_clock_in (optional) }`. `staff_id` from authenticated user.
    *   Logic: Check if staff has an open clock-in entry. If so, return error or auto-clock-out previous. Create new `StaffTimeClockEntries` with `clock_in_time = NOW()`.
    *   Response (201 Created): New time clock entry object.
*   **`POST /api/staff/timeclock/clock-out`**
    *   Permissions: Authenticated Staff (for themselves).
    *   Request: `{ notes_clock_out (optional) }`. `staff_id` from authenticated user.
    *   Logic: Find the latest open `StaffTimeClockEntries` for the staff. If none, error. Update `clock_out_time = NOW()`, calculate `duration_minutes`.
    *   Response (200 OK): Updated time clock entry object.
*   **`GET /api/staff/{staffId}/timeclock-entries`**
    *   Permissions: Admin, or Staff for their own ID.
    *   Query Params: `startDate`, `endDate`, `page`, `limit`.
    *   Response (200 OK): `{ entries: [...], totalPages, currentPage, totalDurationMinutesForPeriod }`.
*   **`POST /api/staff/timeclock-entries` (Admin Manual Entry)**
    *   Permissions: Admin.
    *   Request: `{ staff_id, clock_in_time, clock_out_time, notes_clock_in, notes_clock_out }`.
    *   Logic: `is_manual_entry = true`, `manual_entry_approved_by_user_id = admin_user_id`. Calculate `duration_minutes`.
    *   Response (201 Created): New time clock entry.
*   **`PUT /api/staff/timeclock-entries/{entryId}` (Admin Edit/Approve)**
    *   Permissions: Admin.
    *   Request: Fields to update. If approving a manual entry, can set `manual_entry_approved_by_user_id`.
    *   Response (200 OK): Updated time clock entry.
*   **`DELETE /api/staff/timeclock-entries/{entryId}` (Admin Delete)**
    *   Permissions: Admin.
    *   Response (204 No Content).

## IV. Staff Performance Metrics & Reporting

### 1. Logic (Backend - `PerformanceReportService`)

*   **Data Aggregation:**
    *   **Services Performed:** `COUNT(DISTINCT Appointments.service_id)` or `SUM(TransactionItems.quantity)` where item is a service, grouped by `staff_id`. From `Appointments` (status 'Completed') or `TransactionItems` linked to staff.
    *   **Retail Sales Value/Volume:** `SUM(TransactionItems.price_after_discount)` and `SUM(TransactionItems.quantity)` where `product_id IS NOT NULL`, grouped by `staff_id`.
    *   **Hours Worked:** `SUM(StaffTimeClockEntries.duration_minutes)` for the period.
    *   **Commissions Earned:** `SUM(StaffCommissionsLedger.calculated_commission_amount)` where status is 'approved' or 'paid'.
    *   **Customer Retention (Basic - can be complex):**
        *   Identify customers served by a staff member in a prior period (e.g., 3-6 months ago).
        *   Count how many of those customers had another appointment with the *same staff member* in the current period.
        *   Rate = (Retained Customers / Total Customers from Prior Period) * 100.
    *   **Average Service Time (if tracked per appointment):** More complex, requires start/end times per service instance, not just overall clock-in/out. For now, this might be omitted from "basic".

### 2. API Endpoints (Backend - Admin)

*   **`GET /api/reports/staff-performance/{staffId}`**
    *   Permissions: Admin.
    *   Query Params: `startDate`, `endDate`.
    *   Response (200 OK):
        ```json
        {
          "staffId": 123,
          "staffName": "Jane Doe",
          "period": { "startDate": "YYYY-MM-DD", "endDate": "YYYY-MM-DD" },
          "summary": {
            "totalAppointmentsCompleted": 50,
            "distinctServicesPerformedCount": 15, // Count of unique service types
            "totalRetailUnitsSold": 30,
            "totalRetailSalesValue": 450.75,
            "totalHoursWorkedMinutes": 9600, // Convert to hours in frontend
            "totalCommissionEarned": 850.20 // Sum of 'approved' or 'paid'
          },
          "serviceBreakdown": [ // Optional detailed breakdown
            { "serviceName": "Haircut", "count": 20, "revenueGenerated": 1200.00 },
            { "serviceName": "Coloring", "count": 10, "revenueGenerated": 1500.00 }
          ],
          "productSalesBreakdown": [ // Optional
            { "productName": "Shampoo X", "unitsSold": 5, "revenueGenerated": 100.00 }
          ]
        }
        ```
*   **`GET /api/reports/staff-performance/overview`**
    *   Permissions: Admin.
    *   Query Params: `startDate`, `endDate`.
    *   Response (200 OK): Array of summary objects (similar to `summary` part of the single staff report) for all relevant staff, allowing for comparison.

## V. UI/UX Outlines

### 1. Admin UI

*   **Commission Rules Management (`/admin/commissions/rules`):**
    *   Table of existing rules with filters.
    *   Form (modal/page) to create/edit rules: select staff (optional), service/product (optional), type, value, dates.
*   **Commissions Ledger Viewer (`/admin/commissions/ledger`):**
    *   Table of all ledger entries. Filters by staff, date, status.
    *   Ability to select entries and change status (e.g., "Mark as Approved", "Mark as Paid").
*   **Time Clock Management (`/admin/timeclock`):**
    *   Table of all `StaffTimeClockEntries`. Filters by staff, date.
    *   "Add Manual Entry" button (opens form).
    *   Edit/Delete/Approve actions on entries.
*   **Staff Performance Reports (`/admin/reports/staff`):**
    *   Dropdown to select staff member, date range pickers.
    *   Display of metrics as defined in API response (summary cards, charts for trends, tables for breakdowns).
    *   Comparative overview report.

### 2. Staff UI (Web or Mobile App)

*   **Time Clock (Dashboard or dedicated section):**
    *   Large "Clock In" / "Clock Out" button.
    *   Displays current status (Clocked In since X time / Clocked Out).
    *   Input for notes (optional).
*   **My Time Entries Page:**
    *   List of their own `StaffTimeClockEntries` for a selected period.
*   **My Commissions Page:**
    *   List of their own `StaffCommissionsLedger` entries, filter by date/status.
    *   Summary of pending and paid commissions.

## VI. Deliverables Summary

*   **Database Schemas:** For `StaffCommissionRules`, `StaffCommissionsLedger`, `StaffTimeClockEntries`.
*   **API Designs:**
    *   CRUD for `StaffCommissionRules`.
    *   Viewing/Managing `StaffCommissionsLedger`.
    *   Time Clock operations (`clock-in`, `clock-out`, manual entries).
    *   Staff Performance Reports (`individual`, `overview`).
*   **Logic Outlines:**
    *   Commission calculation (hierarchy, triggers).
    *   Data aggregation for performance metrics.
*   **UI/UX Outlines:** Admin and Staff views for managing and viewing payroll and performance data.

This design provides a comprehensive plan for these Staff Management enhancements.
