# Inventory Management Module Enhancements - Design Document

This document outlines the design for enhancements to the Inventory Management system, including automatic reorder notifications, product expiration tracking (with batching), basic cost analysis, and barcode scanning support. It references `inventory_management_module_design.md`.

## I. Data Model Design (PostgreSQL)

### 1. `Products` Table (Modifications)

*   The existing `current_quantity` field in `Products` will now represent the *total* current quantity across all batches of that product. This will be a calculated sum or kept in sync by triggers/application logic when `ProductBatches` are updated. For simplicity in querying, it's often best to update it via application logic whenever a batch changes.
*   **New/Modified Fields:**
    *   `current_quantity` (INTEGER NOT NULL DEFAULT 0 CHECK (`current_quantity` >= 0)): *This value should always equal the sum of `current_quantity_in_batch` for all active batches of this product.*
    *   `last_checked_for_reorder` (TIMESTAMP WITH TIME ZONE, Nullable): To help optimize reorder job or prevent spamming notifications.

*Conceptual Change Implication:* Direct updates to `Products.current_quantity` should be disallowed or carefully managed; all stock changes should go through batch operations or inventory adjustments that then update batches.

### 2. `ProductBatches` Table (New)

*   Tracks individual shipments/batches of products with specific expiration dates and quantities.
*   **Schema:**
    *   `id` (SERIAL PRIMARY KEY)
    *   `product_id` (INTEGER NOT NULL REFERENCES `Products`(`id`) ON DELETE CASCADE)
    *   `batch_number` (TEXT, Nullable): Supplier's batch number or an internal identifier.
    *   `quantity_received` (INTEGER NOT NULL CHECK (`quantity_received` > 0))
    *   `current_quantity_in_batch` (INTEGER NOT NULL DEFAULT 0 CHECK (`current_quantity_in_batch` >= 0 AND `current_quantity_in_batch` <= `quantity_received`))
    *   `received_date` (DATE NOT NULL DEFAULT CURRENT_DATE)
    *   `expiration_date` (DATE, Nullable): If null, product does not expire.
    *   `supplier_id` (INTEGER, REFERENCES `Suppliers`(`id`) ON DELETE SET NULL, Nullable)
    *   `purchase_price_at_receipt` (DECIMAL(10, 2), Nullable): Cost of goods for this specific batch, for more accurate COGS. If null, can fallback to `Products.purchase_price`.
    *   `created_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)
    *   `updated_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)

*SQL for `ProductBatches` Table:*
```sql
CREATE TABLE ProductBatches (
    id SERIAL PRIMARY KEY,
    product_id INTEGER NOT NULL REFERENCES Products(id) ON DELETE CASCADE,
    batch_number TEXT,
    quantity_received INTEGER NOT NULL CHECK (quantity_received > 0),
    current_quantity_in_batch INTEGER NOT NULL DEFAULT 0 CHECK (current_quantity_in_batch >= 0 AND current_quantity_in_batch <= quantity_received),
    received_date DATE NOT NULL DEFAULT CURRENT_DATE,
    expiration_date DATE, -- Nullable if product does not expire
    supplier_id INTEGER REFERENCES Suppliers(id) ON DELETE SET NULL,
    purchase_price_at_receipt DECIMAL(10, 2) CHECK (purchase_price_at_receipt IS NULL OR purchase_price_at_receipt >= 0),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
    CONSTRAINT fk_product_batch FOREIGN KEY (product_id) REFERENCES Products(id) ON DELETE CASCADE,
    CONSTRAINT fk_supplier_batch FOREIGN KEY (supplier_id) REFERENCES Suppliers(id) ON DELETE SET NULL
);
CREATE INDEX idx_productbatches_product_id ON ProductBatches(product_id);
CREATE INDEX idx_productbatches_expiration_date ON ProductBatches(expiration_date) WHERE expiration_date IS NOT NULL;
CREATE INDEX idx_productbatches_current_quantity ON ProductBatches(current_quantity_in_batch);
```

### 3. `Notifications` Table (New - for In-App)

*   Stores system-generated notifications for admin/staff.
*   **Schema:**
    *   `id` (SERIAL PRIMARY KEY)
    *   `user_id` (INTEGER, REFERENCES `Users`(`id`) ON DELETE CASCADE, Nullable): Recipient user (e.g., admin role group). If NULL, it's a general system notification.
    *   `message` (TEXT NOT NULL)
    *   `type` (TEXT NOT NULL, CHECK (`type` IN ('reorder_alert', 'expiration_alert', 'low_stock_warning', 'general')))
    *   `related_entity_type` (TEXT, Nullable, e.g., "Product", "ProductBatch")
    *   `related_entity_id` (INTEGER, Nullable) -- Link to the product/batch causing the alert
    *   `is_read` (BOOLEAN DEFAULT FALSE NOT NULL)
    *   `created_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)

*SQL for `Notifications` Table:*
```sql
CREATE TABLE Notifications (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES Users(id) ON DELETE CASCADE,
    message TEXT NOT NULL,
    type TEXT NOT NULL CHECK (type IN ('reorder_alert', 'expiration_alert', 'low_stock_warning', 'general')),
    related_entity_type TEXT,
    related_entity_id INTEGER,
    is_read BOOLEAN DEFAULT FALSE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL
);
CREATE INDEX idx_notifications_user_id_is_read ON Notifications(user_id, is_read) WHERE user_id IS NOT NULL;
CREATE INDEX idx_notifications_type ON Notifications(type);
```

## II. Automatic Reorder Notifications

### 1. Logic (Backend - Scheduled Job)

*   **Technology:** `node-cron`. Runs daily.
*   **Job Logic:**
    1.  Queries `Products` table:
        *   Select `id`, `name`, `current_quantity`, `reorder_level`.
        *   Where `reorder_level IS NOT NULL` AND `current_quantity <= reorder_level`.
        *   And `is_active = true` (or similar flag indicating the product is still managed).
        *   Optionally, consider `last_checked_for_reorder` or check against existing unread notifications for the same product to avoid duplicate alerts daily.
    2.  For each product found:
        *   Generate a notification message: e.g., "Product '[Product Name]' (ID: [ID]) is low on stock ([Current Quantity] remaining, reorder level is [Reorder Level])."
        *   **Delivery:**
            *   **In-App:** Create a record in the `Notifications` table. `user_id` could be for a specific inventory manager role or null for general admin view. `type = 'reorder_alert'`, `related_entity_type = 'Product'`, `related_entity_id = product.id`.
            *   **Email (Optional):** Send an email to pre-configured admin/manager email addresses using `EmailService`.

### 2. API (Backend - for In-App Notifications)

*   **`GET /api/notifications`**
    *   Permissions: Admin/Manager roles.
    *   Query Params: `unread` (boolean, optional, default true), `type` (string, optional), `page`, `limit`.
    *   Response (200 OK): `{ notifications: [...], totalPages, currentPage }`.
*   **`PUT /api/notifications/{notificationId}/mark-read`**
    *   Permissions: Admin/Manager (recipient of the notification).
    *   Response (200 OK): Updated notification object or (204 No Content).
*   **`PUT /api/notifications/mark-all-read`**
    *   Permissions: Admin/Manager.
    *   Response (204 No Content).

### 3. UI (Frontend - Admin)

*   **Dashboard/Notification Panel:**
    *   Icon indicating number of unread notifications.
    *   Dropdown/panel listing recent unread notifications. Clicking navigates to the item or marks as read.
*   **Dedicated Notifications Page:** Lists all notifications with filters (type, read/unread).
*   **Inventory Reports Page:**
    *   A section/report "Items Below Reorder Level" that directly queries products meeting this criterion.

## III. Product Expiration Tracking

### 1. API & Logic (Backend)

*   **Receiving Products (Modifying `POST /api/products/{productId}/adjustments` or new endpoint `POST /api/products/receive-batch`):**
    *   The existing `POST /api/products/{productId}/adjustments` could be used if `adjustment_type` is 'purchase_order_received' or 'initial_stock'.
    *   Alternatively, a dedicated endpoint `POST /api/products/receive-batch`:
        *   Request: `{ productId, batchNumber (optional), quantityReceived, expirationDate (optional), supplierId (optional), purchasePriceAtReceipt (optional) }`
        *   Logic:
            1.  Create a new record in `ProductBatches` with `current_quantity_in_batch = quantityReceived`.
            2.  Update `Products.current_quantity` by adding `quantityReceived`.
            3.  Create an `InventoryAdjustments` record: `product_id`, `quantity_change = quantityReceived`, `adjustment_type = 'purchase_order_received'` (or similar), link to `product_batch_id` if possible.
    *   Response: New `ProductBatches` object.
*   **POS/Stock Deduction Logic (Modifying `POST /api/pos/transactions` - from `inventory_management_module_design.md`):**
    *   When a product is sold:
        1.  Identify `productId` and `quantityToDeduct`.
        2.  Implement **FEFO (First Expiring, First Out)** strategy:
            *   Query `ProductBatches` for the given `productId` with `current_quantity_in_batch > 0`, ordered by `expiration_date ASC` (NULLs last or first based on policy for non-expiring items), then `received_date ASC`.
            *   Iterate through batches:
                *   If a batch's `current_quantity_in_batch >= quantityToDeduct`, decrement it, and stop.
                *   If batch's `current_quantity_in_batch < quantityToDeduct`, deduct its entire quantity, update `quantityToDeduct` for the remaining amount, and move to the next batch.
            *   For each batch affected, update its `current_quantity_in_batch`.
        3.  Update `Products.current_quantity` (decrement by total `quantityToDeduct`).
        4.  The `InventoryAdjustments` record for "sale" should reflect the total quantity sold, and potentially could link to which batches were affected if very granular tracking is needed (more complex).
*   **Manual Adjustments (Modifying `POST /api/products/{productId}/adjustments`):**
    *   If `adjustment_type` is 'damage', 'manual_count', 'internal_use':
        *   Request should optionally specify `batchId` if a specific batch is being adjusted.
        *   If `batchId` provided, update that batch's `current_quantity_in_batch`.
        *   If no `batchId`, apply FEFO for deductions or add to a default/newest batch for additions (policy needed).
        *   Update `Products.current_quantity` accordingly.
*   **Scheduled Job for Expiration Alerts (Daily `node-cron` task):**
    *   Queries `ProductBatches`:
        *   Where `expiration_date IS NOT NULL` AND `expiration_date <= NOW() + interval 'X days'` (e.g., X = 30, 60, 90 days).
        *   And `current_quantity_in_batch > 0`.
    *   For each batch found, generate a notification (in-app via `Notifications` table, and/or email):
        *   Message: "Product '[Product Name]' (Batch: [Batch #]) has [Quantity] units expiring on [Expiration Date]."
        *   `type = 'expiration_alert'`, `related_entity_type = 'ProductBatch'`, `related_entity_id = batch.id`.
*   **`GET /api/products/{productId}/batches`**
    *   Permissions: Admin.
    *   Response (200 OK): Array of `ProductBatches` objects for the given product.

### 2. UI (Frontend - Admin)

*   **Product Receiving Form:**
    *   Fields for `productId`, `batchNumber`, `quantityReceived`, `expirationDate`, `supplierId`, `purchasePriceAtReceipt`.
    *   Calls `POST /api/products/receive-batch` or the modified adjustment endpoint.
*   **Product Details View:**
    *   Tab/section to list all batches for the product (`GET /api/products/{productId}/batches`).
    *   Displays batch number, quantity in batch, received date, expiration date.
*   **Inventory Reports Page:**
    *   Section/report "Products Nearing Expiration" (e.g., within next 30/60/90 days).
    *   Section/report "Expired Products" (expiration date has passed, quantity > 0).
*   **Manual Inventory Adjustment Form:**
    *   Option to select a specific batch when adjusting stock for 'damage' or 'manual_count'.

## IV. Basic Cost Analysis

### 1. Logic (Backend)

*   **Product Profit Margin:**
    *   `Products.retail_price`.
    *   `Products.purchase_price` (average/default cost). For more accuracy, average `purchase_price_at_receipt` from `ProductBatches` weighted by quantity, or use FEFO/LIFO costing from batches for COGS (Cost of Goods Sold).
    *   For basic analysis, `Products.purchase_price` is sufficient.
    *   Margin Value: `retail_price - purchase_price`.
    *   Margin Percentage: `((retail_price - purchase_price) / retail_price) * 100`.
*   **COGS for Sales:**
    *   When a product is sold (via POS), the cost of that item is its `purchase_price` (or `purchase_price_at_receipt` from the specific batch it was decremented from, if using batch costing).
    *   `TransactionItems` could store `cost_price_at_sale` for historical accuracy.

### 2. API (Backend)

*   **`GET /api/products/{productId}` and `GET /api/products` (Enhance Existing):**
    *   Add `profit_margin_value` and `profit_margin_percentage` to the response object for each product, calculated using `Products.retail_price` and `Products.purchase_price`.
*   **`GET /api/reports/inventory/profitability` (New)**
    *   Permissions: Admin.
    *   Query Params: `startDate` (YYYY-MM-DD), `endDate` (YYYY-MM-DD), `productId` (optional), `categoryId` (optional).
    *   Logic:
        1.  Fetch `TransactionItems` within the date range, filtered by product/category if provided.
        2.  For each item, get `product_id`, `quantity`, `unit_price` (sale price), and `total_price` (sale total).
        3.  For COGS: Join with `Products` to get `purchase_price`. (More advanced: Join with `ProductBatches` used for that sale to get `purchase_price_at_receipt` if implementing batch costing. For basic, `Products.purchase_price` is fine).
        4.  Aggregate data per product:
            *   `total_units_sold`
            *   `total_revenue` (sum of `TransactionItems.total_price` for that product)
            *   `total_cogs` (sum of `TransactionItems.quantity * Products.purchase_price`)
            *   `total_profit` (`total_revenue - total_cogs`)
    *   Response (200 OK):
        ```json
        {
          "reportTitle": "Inventory Profitability",
          "startDate": "YYYY-MM-DD",
          "endDate": "YYYY-MM-DD",
          "products": [
            {
              "productId": 101,
              "productName": "Luxury Shampoo",
              "totalUnitsSold": 50,
              "averageRetailPrice": 25.00, // Optional: SUM(total_price)/SUM(quantity)
              "averagePurchasePrice": 10.00, // Optional: SUM(cogs)/SUM(quantity)
              "totalRevenue": 1250.00,
              "totalCOGS": 500.00,
              "totalProfit": 750.00,
              "profitMarginPercentage": 60.00 // (totalProfit / totalRevenue) * 100
            }
            // ... more products
          ],
          "summary": {
              "overallTotalRevenue": "...",
              "overallTotalCOGS": "...",
              "overallTotalProfit": "..."
          }
        }
        ```

### 3. UI (Frontend - Admin)

*   **Product List/Details Pages:** Display `profit_margin_value` and `profit_margin_percentage`.
*   **New Report Page (`/admin/reports/inventory-profitability`):**
    *   Filters for date range, product, category.
    *   Table displaying profitability data per product as per API response.
    *   Summary totals.

## V. Barcode Scanning Capabilities

### 1. Hardware Consideration

*   Standard USB or Bluetooth barcode scanners that emulate keyboard input. No special web API integration needed initially. The focus is on UI fields being ready for this input.

### 2. API & Logic (Backend)

*   **`GET /api/products` (Enhance Existing):**
    *   Ensure existing search functionality can efficiently query by `barcode` field.
    *   A specific query param `?barcode={barcode_value}` should return a unique product or an error if not found/multiple found (though barcode should be unique).

### 3. UI (Frontend)

*   **POS Item Selection (`TransactionItemCreator` in `pos_enhancements_design.md`):**
    *   A dedicated input field "Scan Barcode".
    *   This field should auto-focus when the POS screen is active or when a button "Enable Barcode Scan" is clicked.
    *   When a barcode is scanned, the scanner inputs the digits and typically an "Enter" keystroke.
    *   On "Enter" (or `onchange` if scanner doesn't send Enter), the frontend triggers `GET /api/products?barcode={scanned_value}&isSellable=true`.
    *   If a single product is found:
        *   Automatically add it to the transaction cart with quantity 1.
        *   If already in cart, increment quantity.
        *   Play a success sound (optional).
    *   If not found or error: Display message, play error sound (optional).
*   **Admin Product Management (`ProductFormModal`):**
    *   When adding/editing a product, a button "Scan Barcode for Product" next to the `barcode` field.
    *   Clicking it focuses the `barcode` field, allowing the admin to scan the product's barcode directly into the field.
*   **Inventory Adjustments / Stocktaking:**
    *   Similar to POS: a "Scan Barcode" input field to quickly find the product to adjust.
    *   Once product is identified, focus moves to quantity input field.

## VI. Deliverables Summary

*   **Database Schema Changes/Additions:**
    *   Modifications to `Products` table (for total quantity).
    *   New `ProductBatches` table (for expiration and batch tracking).
    *   New `Notifications` table (for in-app alerts).
*   **Scheduled Job Designs:**
    *   Daily job for automatic reorder notifications.
    *   Daily job for product expiration alerts.
*   **API Designs:**
    *   Notifications: `GET /api/notifications`, `PUT /api/notifications/{notificationId}/mark-read`.
    *   Batch Management: `POST /api/products/receive-batch` (or modified adjustments), `GET /api/products/{productId}/batches`.
    *   Profitability: Enhanced `GET /api/products` (to include margin), new `GET /api/reports/inventory/profitability`.
    *   Barcode: Enhanced `GET /api/products` to support efficient lookup by `barcode`.
*   **Logic Outlines:**
    *   FEFO (First Expiring, First Out) for stock deduction from batches.
    *   COGS calculation for profitability report.
*   **UI/UX Considerations:**
    *   Displaying alerts and reports for reorders and expirations.
    *   Managing product batches.
    *   Viewing profitability data.
    *   Integrating barcode scanning into POS, product management, and stocktaking workflows.

This design provides a comprehensive plan for these inventory management enhancements.
