# Advanced Reporting & Analytics - Design Document

This document outlines the design for Advanced Reporting & Analytics features, including advanced KPI dashboards and support for simplified custom report generation. It references existing data models and reporting APIs from previous phases.

## I. Advanced KPI Dashboards

### 1. Key Performance Indicators (KPIs) - Definitions & Calculations

#### A. Revenue Trends

*   **Month-over-Month (MoM) Revenue Growth:**
    *   Calculation: `((Current Month Revenue - Previous Month Revenue) / Previous Month Revenue) * 100%`
    *   Data Source: `Transactions.grand_total`, `Transactions.created_at`.
*   **Year-over-Year (YoY) Revenue Growth:**
    *   Calculation: `((Current Year's Month Revenue - Previous Year's Same Month Revenue) / Previous Year's Same Month Revenue) * 100%`
    *   Data Source: `Transactions.grand_total`, `Transactions.created_at`.
*   **Average Revenue Per Customer (ARPC):**
    *   Calculation: `Total Revenue in Period / Total Unique Customers with Transactions in Period`
    *   Data Source: `Transactions.grand_total`, `Transactions.customer_id`, `Transactions.created_at`.
*   **Revenue by Service Category / Product Category:**
    *   Calculation: `SUM(TransactionItems.price_after_discount)` grouped by `Services.category` or `Products.category`.
    *   Data Source: `TransactionItems`, `Services`, `Products`, `Transactions.created_at`.

#### B. Appointment & Booking Analytics

*   **Booking Lead Time:**
    *   Calculation: Average of (`Appointments.start_time` - `Appointments.created_at`) for appointments booked and occurring within a period.
    *   Data Source: `Appointments.start_time`, `Appointments.created_at`.
*   **Peak Booking Times/Days:**
    *   Calculation: Count of appointments grouped by `EXTRACT(DOW FROM start_time)` (day of week) and `EXTRACT(HOUR FROM start_time)`.
    *   Data Source: `Appointments.start_time`.
*   **Online vs. Manual Booking Ratio:**
    *   Calculation: `(Count of Appointments created via API by Customer / Count of Appointments created via API by Staff/Admin)`. Requires a field in `Appointments` like `booked_by_channel` ('online_customer', 'manual_staff').
    *   Data Source: `Appointments.booked_by_channel` (New field needed in `Appointments`).
*   **Appointment Fill Rate:**
    *   Calculation: `(Total Duration of Booked Appointments for Staff / Total Duration of Available Work Time for Staff) * 100%` for a period.
    *   Data Source: `Appointments.duration_minutes` (or calculated from start/end time), `StaffRecurringAvailability`, `StaffAvailabilityOverrides`. Complex, requires knowing actual staff working hours.
*   **Cancellation Rate:**
    *   Calculation: `(Count of Appointments with status 'cancelled_by_customer' or 'cancelled_by_salon' / Total Appointments Scheduled for Period) * 100%`
    *   Data Source: `Appointments.status`, `Appointments.created_at` (for when it was scheduled).
*   **No-Show Rate:**
    *   Calculation: `(Count of Appointments with status 'no_show' / Total Appointments Scheduled for Period excluding cancellations) * 100%`
    *   Data Source: `Appointments.status`, `Appointments.created_at`.

#### C. Customer Retention & Loyalty

*   **Customer Retention Rate (CRR):**
    *   Calculation (Cohort-based example for a period, e.g., monthly):
        1.  Customers with transactions in Period 1 (C1).
        2.  Customers from C1 who also had transactions in Period 2 (C2_retained).
        3.  CRR = `(C2_retained / C1) * 100%`.
    *   Data Source: `Transactions.customer_id`, `Transactions.created_at`.
*   **New vs. Returning Customer Ratio:**
    *   Calculation: For a period, (Count of Customers whose first transaction is in this period) / (Count of Customers whose first transaction was before this period but had a transaction in this period).
    *   Data Source: `Transactions.customer_id`, `Transactions.created_at`. Requires finding min(`created_at`) per `customer_id`.
*   **Customer Lifetime Value (CLTV - Basic):**
    *   Calculation: `ARPC * Average Customer Lifespan`. Average Customer Lifespan is complex to calculate accurately (e.g., (Sum of (Last_transaction_date - First_transaction_date) for all customers) / Total Customers). For basic, can use a fixed period (e.g., 12 or 24 months) or average time between first/last transaction for *active* customers.
    *   Data Source: ARPC, `Transactions.customer_id`, `Transactions.created_at`.
*   **Loyalty Program Engagement:**
    *   Points Earned/Redeemed Ratio: `SUM(LoyaltyPointsLedger.points_redeemed) / SUM(LoyaltyPointsLedger.points_earned)` in a period.
    *   % of Customers Using Loyalty: `(Count of Unique Customers in LoyaltyPointsLedger / Total Unique Customers with Transactions) * 100%` in a period.
    *   Data Source: `LoyaltyPointsLedger`, `Transactions`.

#### D. Service & Product Performance

*   **Service Popularity Heatmaps:**
    *   Visual representation (e.g., table with color intensity) showing `COUNT(Appointments.id)` or `SUM(TransactionItems.quantity where service_id is not null)` grouped by `Services.name` and `Users.first_name || ' ' || Users.last_name` (Staff), possibly also by day/time blocks.
    *   Data Source: `TransactionItems`, `Appointments`, `Services`, `Users` (Staff).
*   **Product Sales Velocity:**
    *   Calculation: `Total Units of Product Sold in Period / Number of Days in Period`.
    *   Data Source: `TransactionItems.quantity` (where `product_id` is not null), `Products.name`, `Transactions.created_at`.
*   **Service/Product Cross-Sell Analysis:**
    *   Calculation: Identify pairs of services or products frequently appearing in the same `Transaction`. E.g., "Customers who booked 'Service A' also booked 'Service B' X% of the time."
    *   Data Source: `TransactionItems`, `Transactions.id`. Requires analyzing co-occurrence.

#### E. Staff Performance (Beyond Basic)

*   **Client Retention Rate per Staff:**
    *   Similar to overall CRR, but filtered for appointments/transactions handled by a specific staff member.
    *   Data Source: `Transactions.customer_id`, `Transactions.created_at`, `TransactionItems.staff_id` (or `Appointments.staff_id`).
*   **Average Client Rating per Staff (Requires Rating System - Future Feature):**
    *   If a rating system (e.g., 1-5 stars post-appointment) is added: `SUM(Ratings.rating_value) / COUNT(Ratings.id)` grouped by `staff_id`.
    *   Data Source: New `Ratings` table.
*   **Upselling/Cross-selling performance per Staff:**
    *   Calculation:
        *   Average number of items (services + products) per transaction handled by staff.
        *   Average transaction value per staff.
        *   % of transactions with more than one distinct service or with both service and product.
    *   Data Source: `TransactionItems`, `Transactions`, `TransactionItems.staff_id`.

### 2. Backend API Enhancements/New Endpoints

APIs should accept `startDate` and `endDate` query parameters for period definition. Responses should be structured for easy consumption by charting libraries.

*   **`GET /api/reports/kpi/revenue-growth`**
    *   Response: `{ "momGrowth": X, "yoyGrowth": Y, "currentMonthRevenue": Z, "previousMonthRevenue": A, "previousYearSameMonthRevenue": B }`
*   **`GET /api/reports/kpi/arpc`**
    *   Response: `{ "arpc": X, "totalRevenue": Y, "uniqueCustomers": Z }`
*   **`GET /api/reports/kpi/revenue-by-category`**
    *   Response: `{ "services": [{ "category": "Hair", "revenue": X }, ...], "products": [{ "category": "Shampoo", "revenue": Y }, ...] }`
*   **`GET /api/reports/kpi/booking-analytics`**
    *   Response: `{ "avgLeadTimeHours": X, "peakBookingDays": [{"dayOfWeek": 1, "count": Y}, ...], "peakBookingHours": [{"hour": 9, "count": Z}, ...], "onlineVsManualRatio": A, "cancellationRate": B, "noShowRate": C }` (Fill rate is complex, might be separate).
*   **`GET /api/reports/kpi/appointment-fill-rate`**
    *   Query Params: `staffId` (optional), `date` (for specific day) or `startDate`/`endDate`.
    *   Response: `{ "staffId": S, "staffName": "...", "totalBookedMinutes": M1, "totalAvailableMinutes": M2, "fillRatePercentage": P }` or array if for multiple staff.
*   **`GET /api/reports/kpi/customer-retention`**
    *   Query Params: `periodType` ('monthly', 'quarterly').
    *   Response: `{ "retentionRate": X, "newVsReturningRatio": { "new": Y, "returning": Z } }` (CLTV might be too complex for direct API, or a simplified version).
*   **`GET /api/reports/kpi/loyalty-engagement`**
    *   Response: `{ "pointsEarnRedeemRatio": X, "percentageCustomersUsingLoyalty": Y }`
*   **`GET /api/reports/kpi/service-product-performance`**
    *   Query Params: `staffId` (optional for heatmap), `serviceId` (optional), `productId` (optional).
    *   Response: `{ "servicePopularity": [ { "serviceName": S, "staffName": T, "bookingCount": C, "timeBlock": "9-12" }, ... ], "productSalesVelocity": [ { "productName": P, "velocity": V }, ... ], "crossSellInsights": [ /* TBD structure, e.g., { "itemA": "ServiceX", "itemB": "ProductY", "coOccurrenceCount": N } */ ] }`
*   **`GET /api/reports/kpi/staff-performance-advanced`**
    *   Query Params: `staffId`.
    *   Response: `{ "staffId": S, "staffName": "...", "clientRetentionRate": X, "avgItemsPerTransaction": Y, "avgTransactionValue": Z, ... }`

### 3. UI (Frontend - Admin Dashboard)

*   **Dedicated "Analytics" Section:**
    *   **Overview Dashboard:** Key headline KPIs (MoM Revenue, ARPC, Retention Rate) using scorecards.
    *   **Sub-sections/Dashboards:** Revenue, Appointments, Customers, Service/Product, Staff.
*   **Visualizations:**
    *   Line charts for trends (Revenue MoM/YoY, ARPC over time).
    *   Bar charts for comparisons (Revenue by Category, Peak Times/Days, New vs. Returning).
    *   Pie charts for ratios (Online vs. Manual, Loyalty Engagement).
    *   Scorecards for single important numbers (Fill Rate, Cancellation Rate, No-Show Rate).
    *   Tables with conditional formatting (heatmaps) for Service Popularity.
*   **Filters:** Global date range filter, and specific filters per dashboard/chart (staff, category).
*   **Interactivity:** Clicking on a chart segment might drill down or filter other charts on the dashboard.
*   **Export:** Option to export chart image (PNG) or underlying data (CSV).

## II. Simplified Custom Report Generation

### 1. Concept

Allow admins to select a primary data source, choose fields, apply basic filters, and view/export results. This avoids hardcoding every possible report variation.

### 2. Backend API (`POST /api/reports/custom` - use POST for complex query body)

*   **Request Body:**
    ```json
    {
      "dataSource": "Appointments", // "Transactions", "Customers", "Services", "Products", "TransactionItems", "LoyaltyPointsLedger"
      "fields": [ // Array of field names. Can include related fields using dot notation (e.g., "customer.first_name")
        "start_time",
        "status",
        "service.name as service_name", // Using alias for clarity
        "staff.first_name as staff_first_name",
        "customer.email as customer_email",
        "transaction_item.total_price as item_revenue" // If dataSource is Appointments and linking to TransactionItems
      ],
      "filters": [ // Array of filter objects
        { "field": "appointments.start_time", "operator": "between", "value": ["YYYY-MM-DD", "YYYY-MM-DD"] },
        { "field": "appointments.status", "operator": "equals", "value": "Completed" },
        { "field": "service.category", "operator": "in", "value": ["Hair Styling", "Nail Care"] }
      ],
      "joins": [ // Optional: Define explicit joins if not automatically inferred by field notation
          // { "from_table": "Appointments", "join_table": "Users", "as": "customer", "on": "Appointments.customer_id = customer.id" },
          // { "from_table": "Appointments", "join_table": "Services", "as": "service", "on": "Appointments.service_id = service.id" }
      ],
      "sortBy": "appointments.start_time",
      "sortOrder": "desc",
      "page": 1,
      "limit": 50
    }
    ```
*   **Logic (Backend - `CustomReportService`):**
    1.  **Validation & Sanitization (CRITICAL):**
        *   Whitelist `dataSource` values.
        *   Validate each field in `fields` against a predefined list of allowed fields for the `dataSource` and its accessible relations (to prevent arbitrary table/column access).
        *   Sanitize aliases.
        *   Whitelist `operator` values (e.g., "equals", "notEquals", "in", "between", "like", "gt", "lt", "gte", "lte").
        *   Sanitize `value` based on expected type for the field and operator.
        *   Validate `sortBy` field.
    2.  **Dynamic SQL Query Building:**
        *   Use a query builder library (e.g., Knex.js, Sequelize ORM query interface) to construct the SQL query safely. Avoid direct string concatenation with user inputs.
        *   Handle joins: Automatically infer joins based on dot notation in `fields` (e.g., `customer.email` from `Appointments` implies a join to `Users as customer`). Or use explicit `joins` array for more complex scenarios.
        *   Apply filters from the `filters` array.
        *   Apply sorting and pagination.
    3.  Execute query.
    *   **Response (200 OK):**
        ```json
        {
          "reportName": "Custom Appointments Report", // Potentially generated or user-defined
          "columns": [ // Derived from selected fields for table header
            { "key": "start_time", "label": "Appointment Time" },
            { "key": "status", "label": "Status" },
            { "key": "service_name", "label": "Service" },
            // ...
          ],
          "data": [ // Array of result objects
            { "start_time": "...", "status": "Completed", "service_name": "Haircut", ... },
            // ...
          ],
          "totalPages": X,
          "currentPage": Y,
          "totalRecords": Z
        }
        ```

### 3. UI (Frontend - Admin `/admin/reports/custom-builder`)

*   **Report Builder Interface:**
    *   **Data Source Selection:** Dropdown for `dataSource`.
    *   **Field Selection:**
        *   Multi-select list or tree view (grouped by table for related fields like "Customer > Email") for `fields`.
        *   Dynamically updates based on selected `dataSource`.
        *   Allows defining aliases (e.g., "Service Name" for `service.name`).
    *   **Filter Builder:**
        *   Interface to add multiple filter rows.
        *   Each row: Dropdown for field (from selected `dataSource` and common relations), dropdown for operator, input for value(s).
    *   **Sort Options:** Dropdown for `sortBy` field, toggle for `sortOrder`.
    *   **Button "Preview Report" / "Generate Report"**: Calls `POST /api/reports/custom`.
*   **Report Results View:**
    *   Displays results in a dynamic, paginated table using the `columns` and `data` from API response.
    *   **Button "Export to CSV"**: Client-side generation from JSON data or server-side if data is too large.
*   **(Optional) Save/Load Report Configurations:**
    *   Ability to name and save the current `dataSource`, `fields`, `filters`, `sortBy` configuration.
    *   List of saved reports to quickly re-run. (Requires new DB tables and APIs for saving/loading report definitions).

### 4. Security Considerations for Custom Reports

*   **Strict Whitelisting:** Only allow predefined data sources, fields, and operators.
*   **Parameterized Queries:** Use a query builder or ORM that ensures parameterized queries to prevent SQL injection.
*   **Input Sanitization/Validation:** Thoroughly validate all inputs, especially values used in filters.
*   **Resource Limiting:** Implement limits on query complexity, result set size, or execution time if possible, to prevent abuse or performance degradation.
*   **Permissions:** Ensure only authorized roles (e.g., Admin) can access this feature.

## III. Deliverables Summary

*   **List of Defined Advanced KPIs:** Details of calculations and data sources for revenue, appointment, customer, service/product, and staff KPIs.
*   **API Designs for KPIs:** New/enhanced API endpoints (e.g., `/api/reports/kpi/...`) with request parameters and expected response structures.
*   **UI/UX Concepts for KPI Dashboards:** Description of the "Analytics" section, visualization types, filters, and interactivity.
*   **Design for Simplified Custom Report Generation:**
    *   Concept: User selects data source, fields, filters; views/exports results.
    *   API: `POST /api/reports/custom` with detailed request (dataSource, fields, filters, sorting, pagination) and response structure (columns, data).
    *   UI/UX: Report builder interface (data source/field/filter selection) and results display (table, CSV export).
    *   Security: Emphasis on whitelisting, parameterized queries, and input validation.

This design provides a comprehensive plan for Advanced Reporting & Analytics features.
