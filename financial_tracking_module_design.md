# Basic Financial Tracking Module - Design Document

This document outlines the design for Basic Financial Tracking features, including revenue reporting from POS data and a system for logging operational expenses. It references `pos_module_design.md` and `project_setup_plan.md`.

## I. Data Model Design (PostgreSQL)

1.  **`Expenses` Table:**
    *   Stores records of operational expenses.
    *   **Schema:**
        *   `id` (SERIAL PRIMARY KEY)
        *   `expense_date` (DATE NOT NULL)
        *   `category` (TEXT NOT NULL, e.g., "Rent", "Utilities", "Stock Purchase", "Marketing", "Salaries", "Other")
        *   `description` (TEXT NOT NULL)
        *   `amount` (DECIMAL(10, 2) NOT NULL CHECK (`amount` > 0))
        *   `supplier_id` (INTEGER, REFERENCES `Suppliers`(`id`) ON DELETE SET NULL, Nullable)
        *   `created_by_user_id` (INTEGER NOT NULL, REFERENCES `Users`(`id`) ON DELETE RESTRICT) -- User who logged the expense
        *   `receipt_url` (TEXT, Nullable) -- URL to a scanned receipt image/PDF
        *   `created_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)
        *   `updated_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)

    *SQL for `Expenses` Table:*
    ```sql
    CREATE TABLE Expenses (
        id SERIAL PRIMARY KEY,
        expense_date DATE NOT NULL,
        category TEXT NOT NULL,
        description TEXT NOT NULL,
        amount DECIMAL(10, 2) NOT NULL CHECK (amount > 0),
        supplier_id INTEGER REFERENCES Suppliers(id) ON DELETE SET NULL,
        created_by_user_id INTEGER NOT NULL REFERENCES Users(id) ON DELETE RESTRICT,
        receipt_url TEXT,
        created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
        updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
        CONSTRAINT fk_expense_supplier FOREIGN KEY (supplier_id) REFERENCES Suppliers(id) ON DELETE SET NULL,
        CONSTRAINT fk_expense_user FOREIGN KEY (created_by_user_id) REFERENCES Users(id) ON DELETE RESTRICT
    );
    CREATE INDEX idx_expenses_expense_date ON Expenses(expense_date);
    CREATE INDEX idx_expenses_category ON Expenses(category);
    CREATE INDEX idx_expenses_supplier_id ON Expenses(supplier_id);
    ```

2.  **Existing Tables Referenced:**
    *   `Transactions` and `TransactionItems` (from `pos_module_design.md`): Used for revenue calculations.
    *   `Suppliers` (from `inventory_management_module_design.md`): Linked to expenses.
    *   `Users` (from `project_setup_plan.md`): Linked to `created_by_user_id` in `Expenses`.

## II. Backend API Development (Node.js/Express & PostgreSQL)

Appropriate authentication (`ensureAuthenticated`) and authorization (`ensureRole`) middleware will be applied. Typically, these are Admin/Manager roles.

### 1. Expense CRUD API Endpoints

*   **`POST /api/expenses`**
    *   Permissions: Admin, or Staff with specific financial permissions.
    *   Request Body:
        ```json
        {
          "expense_date": "YYYY-MM-DD",
          "category": "Utilities",
          "description": "Monthly electricity bill",
          "amount": 150.75,
          "supplier_id": null, // Optional
          "receipt_url": "https://example.com/receipt.pdf" // Optional, URL obtained from upload step
        }
        ```
        *(Note: `created_by_user_id` will be set based on the authenticated user making the request).*
    *   Response (201 Created): New expense object.
*   **`GET /api/expenses`**
    *   Permissions: Admin, or Staff with specific financial permissions.
    *   Query Parameters:
        *   `startDate` (YYYY-MM-DD)
        *   `endDate` (YYYY-MM-DD)
        *   `category` (string)
        *   `supplierId` (integer)
        *   `page` (integer), `limit` (integer) for pagination.
    *   Response (200 OK): `{ expenses: [...], totalPages, currentPage }`.
*   **`GET /api/expenses/{expenseId}`**
    *   Permissions: Admin, or Staff with specific financial permissions.
    *   Response (200 OK): Single expense object or 404 Not Found.
*   **`PUT /api/expenses/{expenseId}`**
    *   Permissions: Admin, or Staff with specific financial permissions.
    *   Request Body: Fields to update (similar to POST).
    *   Response (200 OK): Updated expense object or 404 Not Found.
*   **`DELETE /api/expenses/{expenseId}`**
    *   Permissions: Admin, or Staff with specific financial permissions.
    *   Logic: Standard delete. Consider implications if linked to other financial reconciliation processes (unlikely for basic tracking).
    *   Response (204 No Content) or 404 Not Found.

*   **Receipt Upload API (Conceptual - if managing uploads directly):**
    *   Similar to photo upload in CRM enhancements:
        *   **`POST /api/expenses/receipts/presigned-url`**: Client requests a pre-signed URL (e.g., for S3) to upload the receipt file.
            *   Request: `{ "fileName": "receipt.pdf", "fileType": "application/pdf" }`
            *   Response: `{ "uploadUrl": "...", "receiptUrl": "final_storage_url" }`
        *   The client uploads the file to the `uploadUrl`.
        *   The `receiptUrl` obtained is then passed when creating/updating an expense via `POST /api/expenses` or `PUT /api/expenses/{expenseId}`.

### 2. Revenue Reporting API Endpoints

*   **`GET /api/reports/revenue/summary`**
    *   Permissions: Admin, or Staff with specific financial permissions.
    *   Request Query Params:
        *   `startDate` (YYYY-MM-DD, required)
        *   `endDate` (YYYY-MM-DD, required)
        *   `groupBy` (string, optional, e.g., "day", "week", "month", "serviceName", "productCategory", "staffName"). Default could be "day" if not provided or "none" for a single total.
    *   Logic:
        *   Queries `Transactions` and `TransactionItems` tables.
        *   `final_amount` from `Transactions` is typically used for total revenue.
        *   For `groupBy` serviceName/productCategory/staffName, it aggregates `TransactionItems.total_price`.
        *   SQL `SUM()`, `GROUP BY`, and date functions will be heavily used.
    *   Response (200 OK): Structured JSON.
        *   **Example for `groupBy=day`:**
            ```json
            {
              "reportTitle": "Daily Revenue Summary",
              "startDate": "YYYY-MM-DD",
              "endDate": "YYYY-MM-DD",
              "totalRevenue": 1250.75,
              "totalTransactions": 15,
              "breakdown": [
                { "period": "YYYY-MM-DD", "revenue": 600.50, "transactions": 8 },
                { "period": "YYYY-MM-DD", "revenue": 650.25, "transactions": 7 }
              ]
            }
            ```
        *   **Example for `groupBy=serviceName`:**
            ```json
            {
              "reportTitle": "Revenue by Service",
              "startDate": "YYYY-MM-DD",
              "endDate": "YYYY-MM-DD",
              "totalRevenueFromServices": 800.00, // Sum of revenue from items identified as services
              "breakdown": [
                { "serviceName": "Women's Haircut", "revenue": 400.00, "count": 8 },
                { "serviceName": "Manicure", "revenue": 250.00, "count": 5 },
                { "serviceName": "Basic Facial", "revenue": 150.00, "count": 2 }
              ]
            }
            ```
        *   **Example for `groupBy=productCategory` (assuming products have categories):**
            ```json
            {
              "reportTitle": "Revenue by Product Category",
              "startDate": "YYYY-MM-DD",
              "endDate": "YYYY-MM-DD",
              "totalRevenueFromProducts": 450.75, // Sum of revenue from items identified as products
              "breakdown": [
                { "productCategory": "Hair Care", "revenue": 300.00, "count": 10 },
                { "productCategory": "Skin Care", "revenue": 150.75, "count": 5 }
              ]
            }
            ```
        *   **Example for `groupBy=staffName`:**
            ```json
            {
              "reportTitle": "Revenue by Staff Member",
              "startDate": "YYYY-MM-DD",
              "endDate": "YYYY-MM-DD",
              "totalRevenue": 1250.75,
              "breakdown": [
                { "staffName": "Jane Doe", "revenue": 700.00, "transactionsProcessedOrItemsSold": 10 },
                { "staffName": "John Smith", "revenue": 550.75, "transactionsProcessedOrItemsSold": 5 }
              ]
            }
            ```
            *(Note: "Revenue by Staff" can be complex - is it based on who processed the transaction or who performed/sold each item? The latter is more accurate for service/item credit and requires joining `TransactionItems` with `Users` via `staff_id`).*

*   **`GET /api/reports/revenue/details`**
    *   Permissions: Admin, or Staff with specific financial permissions.
    *   Request Query Params: `startDate`, `endDate`, `serviceId` (optional), `productId` (optional), `staffId` (optional), `page`, `limit`.
    *   Logic: Returns a paginated list of individual transactions (or transaction items) that match the filter criteria for the given period. This is essentially an enhanced version of `GET /api/pos/transactions` but could be specifically tailored for financial auditing (e.g., always joining item details).
    *   Response (200 OK):
        ```json
        {
          "transactions": [ /* array of detailed transaction objects, including items */ ],
          "totalPages": 5,
          "currentPage": 1
        }
        ```

### 3. Profit/Loss Report API Endpoint (Basic)

*   **`GET /api/reports/profit-loss/summary`**
    *   Permissions: Admin.
    *   Request Query Params: `startDate` (YYYY-MM-DD, required), `endDate` (YYYY-MM-DD, required).
    *   Logic:
        1.  Calculate `totalRevenue`: Sum of `final_amount` from `Transactions` within the date range.
        2.  Calculate `totalExpenses`: Sum of `amount` from `Expenses` where `expense_date` is within the date range.
        3.  Calculate `netProfit = totalRevenue - totalExpenses`.
    *   Response (200 OK):
        ```json
        {
          "reportTitle": "Profit & Loss Summary",
          "startDate": "YYYY-MM-DD",
          "endDate": "YYYY-MM-DD",
          "totalRevenue": 5000.00,
          "totalExpenses": 1500.00,
          "netProfit": 3500.00
        }
        ```

## III. Frontend Outline (React)

### 1. Expense Management Components (Admin/Staff with permissions - e.g., `/admin/financials/expenses`)

*   **`ExpenseListPage` Component:**
    *   Route: `/admin/financials/expenses`.
    *   Uses `GET /api/expenses` to fetch and display data.
    *   **`ExpenseFilters` Sub-Component:** Inputs for date range, category dropdown, supplier dropdown.
    *   **`ExpenseTable` Sub-Component:**
        *   Displays expenses: Date, Category, Description, Amount, Supplier, Actions (Edit, Delete).
        *   Clicking "Edit" opens `ExpenseFormModal` with expense data.
        *   "Delete" button calls `DELETE /api/expenses/{expenseId}`.
    *   **Button "Add New Expense":** Opens `ExpenseFormModal`.
    *   Pagination controls.
*   **`ExpenseFormModal` Component (or dedicated page):**
    *   Form for creating/editing an expense.
    *   Fields: `expense_date` (date picker), `category` (dropdown or text input with suggestions), `description` (textarea), `amount` (number input), `supplier_id` (optional dropdown of suppliers), `receipt_url` (text input, or file input for direct upload if implemented).
    *   If implementing direct receipt upload:
        1.  File input for receipt.
        2.  On file selection, call `POST /api/expenses/receipts/presigned-url`.
        3.  Upload file to S3 using the received URL.
        4.  Store the final `receiptUrl` in the form state to be submitted with other expense data.
    *   On submit, calls `POST /api/expenses` or `PUT /api/expenses/{expenseId}`.

### 2. Revenue Reporting Components (Admin/Staff with permissions - e.g., `/admin/reports/revenue`)

*   **`RevenueReportPage` Component:**
    *   Route: `/admin/reports/revenue`.
    *   **`ReportFilters` Sub-Component:**
        *   Date range selectors (`startDate`, `endDate`).
        *   Dropdown for `groupBy` option (Day, Week, Month, Service, Product Category, Staff).
        *   "Generate Report" button.
    *   **`ReportDisplay` Sub-Component:**
        *   Displays results from `GET /api/reports/revenue/summary`.
        *   Shows key metrics: Total Revenue, Total Transactions.
        *   **Charts:** Uses a charting library (e.g., `Chart.js`, `Recharts`) to visualize breakdown data (e.g., bar chart for revenue over time, pie chart for category/service breakdown).
        *   **Data Table:** Displays the breakdown data in tabular format below the chart.
    *   (Optional) Link/button to view detailed transactions (`GET /api/reports/revenue/details` or a pre-filtered transaction list).

### 3. Profit & Loss Report Component (Admin - e.g., `/admin/reports/profit-loss`)

*   **`ProfitLossReportPage` Component:**
    *   Route: `/admin/reports/profit-loss`.
    *   **`DateRangeFilter` Sub-Component:** For `startDate`, `endDate`.
    *   "Generate Report" button.
    *   **`ProfitLossDisplay` Sub-Component:**
        *   Displays results from `GET /api/reports/profit-loss/summary`.
        *   Clearly shows: Total Revenue, Total Expenses, Net Profit.
        *   Could include simple bar chart comparing revenue vs. expenses.

## IV. Deliverables Summary

*   **Database Schema:** SQL for the `Expenses` table.
*   **Detailed API Endpoint Definitions:**
    *   Expense CRUD APIs: Request/response structures, including receipt upload considerations.
    *   Revenue Reporting APIs (`summary`, `details`): Request/response structures for various `groupBy` options.
    *   Profit/Loss Report API (`summary`): Request/response structure.
*   **Outline of React Components:**
    *   For managing expenses (`ExpenseListPage`, `ExpenseFormModal`).
    *   For viewing revenue reports (`RevenueReportPage` with filters and chart/table display).
    *   For viewing profit/loss reports (`ProfitLossReportPage`).

This design provides a comprehensive plan for the Basic Financial Tracking features.
