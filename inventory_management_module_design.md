# Inventory Management Module - API, Database, and Frontend Design

This document outlines the design for a Basic Inventory Management system, including product and supplier tracking, and integration with the Point of Sale (POS) system. It references `pos_module_design.md` and `project_setup_plan.md`.

## I. Data Model Design (PostgreSQL)

1.  **`Suppliers` Table:**
    *   Stores information about product suppliers.
    *   **Schema:**
        *   `id` (SERIAL PRIMARY KEY)
        *   `name` (TEXT NOT NULL UNIQUE)
        *   `contact_person` (TEXT, Nullable)
        *   `email` (TEXT, Nullable)
        *   `phone_number` (TEXT, Nullable)
        *   `address` (TEXT, Nullable)
        *   `notes` (TEXT, Nullable)
        *   `created_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)
        *   `updated_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)

    *SQL for `Suppliers` Table:*
    ```sql
    CREATE TABLE Suppliers (
        id SERIAL PRIMARY KEY,
        name TEXT NOT NULL UNIQUE,
        contact_person TEXT,
        email TEXT,
        phone_number TEXT,
        address TEXT,
        notes TEXT,
        created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
        updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL
    );
    ```

2.  **`Products` Table:**
    *   Stores details about individual products, including stock levels.
    *   **Schema:**
        *   `id` (SERIAL PRIMARY KEY)
        *   `name` (TEXT NOT NULL)
        *   `sku` (TEXT UNIQUE, Nullable) -- Stock Keeping Unit
        *   `barcode` (TEXT UNIQUE, Nullable)
        *   `description` (TEXT, Nullable)
        *   `category` (TEXT, Nullable, e.g., "Shampoo", "Styling", "Skincare")
        *   `supplier_id` (INTEGER, REFERENCES `Suppliers`(`id`) ON DELETE SET NULL, Nullable)
        *   `purchase_price` (DECIMAL(10, 2), Nullable) -- Price paid to supplier
        *   `retail_price` (DECIMAL(10, 2) NOT NULL CHECK (`retail_price` >= 0)) -- Price sold to customer
        *   `current_quantity` (INTEGER NOT NULL DEFAULT 0 CHECK (`current_quantity` >= 0))
        *   `reorder_level` (INTEGER, Nullable CHECK (`reorder_level` IS NULL OR `reorder_level` >= 0)) -- Threshold for reorder reminder
        *   `is_sellable` (BOOLEAN DEFAULT TRUE NOT NULL) -- Is it a retail product?
        *   `is_professional_use_only` (BOOLEAN DEFAULT FALSE NOT NULL) -- For salon internal use, not for retail
        *   `created_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)
        *   `updated_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)

    *SQL for `Products` Table:*
    ```sql
    CREATE TABLE Products (
        id SERIAL PRIMARY KEY,
        name TEXT NOT NULL,
        sku TEXT UNIQUE,
        barcode TEXT UNIQUE,
        description TEXT,
        category TEXT,
        supplier_id INTEGER REFERENCES Suppliers(id) ON DELETE SET NULL,
        purchase_price DECIMAL(10, 2) CHECK (purchase_price IS NULL OR purchase_price >= 0),
        retail_price DECIMAL(10, 2) NOT NULL CHECK (retail_price >= 0),
        current_quantity INTEGER NOT NULL DEFAULT 0 CHECK (current_quantity >= 0),
        reorder_level INTEGER CHECK (reorder_level IS NULL OR reorder_level >= 0),
        is_sellable BOOLEAN DEFAULT TRUE NOT NULL,
        is_professional_use_only BOOLEAN DEFAULT FALSE NOT NULL,
        created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
        updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
        CONSTRAINT fk_supplier FOREIGN KEY (supplier_id) REFERENCES Suppliers(id) ON DELETE SET NULL
    );
    CREATE INDEX idx_products_category ON Products(category);
    CREATE INDEX idx_products_supplier_id ON Products(supplier_id);
    CREATE INDEX idx_products_is_sellable ON Products(is_sellable);
    ```

3.  **`InventoryAdjustments` Table:**
    *   Tracks changes to product stock levels for auditing and history.
    *   **Schema:**
        *   `id` (SERIAL PRIMARY KEY)
        *   `product_id` (INTEGER NOT NULL, REFERENCES `Products`(`id`) ON DELETE CASCADE)
        *   `adjustment_type` (TEXT NOT NULL, CHECK (`adjustment_type` IN ('initial_stock', 'manual_count', 'damage', 'return_customer', 'return_supplier', 'purchase_order_received', 'sale', 'internal_use', 'other')))
        *   `quantity_change` (INTEGER NOT NULL) -- Can be positive (stock in) or negative (stock out)
        *   `reason` (TEXT, Nullable)
        *   `adjusted_by_user_id` (INTEGER, REFERENCES `Users`(`id`) ON DELETE SET NULL, Nullable) -- Staff member who made/recorded the adjustment
        *   `created_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)

    *SQL for `InventoryAdjustments` Table:*
    ```sql
    CREATE TABLE InventoryAdjustments (
        id SERIAL PRIMARY KEY,
        product_id INTEGER NOT NULL REFERENCES Products(id) ON DELETE CASCADE,
        adjustment_type TEXT NOT NULL CHECK (adjustment_type IN ('initial_stock', 'manual_count', 'damage', 'return_customer', 'return_supplier', 'purchase_order_received', 'sale', 'internal_use', 'other')),
        quantity_change INTEGER NOT NULL,
        reason TEXT,
        adjusted_by_user_id INTEGER REFERENCES Users(id) ON DELETE SET NULL,
        created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
        CONSTRAINT fk_product_adjustment FOREIGN KEY (product_id) REFERENCES Products(id) ON DELETE CASCADE,
        CONSTRAINT fk_user_adjustment FOREIGN KEY (adjusted_by_user_id) REFERENCES Users(id) ON DELETE SET NULL
    );
    CREATE INDEX idx_inventoryadjustments_product_id ON InventoryAdjustments(product_id);
    CREATE INDEX idx_inventoryadjustments_adjustment_type ON InventoryAdjustments(adjustment_type);
    ```

## II. Backend API Development (Node.js/Express & PostgreSQL)

All Admin endpoints protected by `ensureAuthenticated` and `ensureRole(['admin'])`. Staff access for POS is handled by existing POS endpoint permissions.

### 1. Supplier CRUD API Endpoints (Admin)

*   **`POST /api/suppliers`**
    *   Request: `{ name, contact_person, email, phone_number, address, notes }`
    *   Response (201 Created): New supplier object.
*   **`GET /api/suppliers`**
    *   Query Params: `search` (for name), `page`, `limit`.
    *   Response (200 OK): `{ suppliers: [...], totalPages, currentPage }`.
*   **`GET /api/suppliers/{supplierId}`**
    *   Response (200 OK): Single supplier object or 404 Not Found.
*   **`PUT /api/suppliers/{supplierId}`**
    *   Request: `{ name, contact_person, ... }` (fields to update)
    *   Response (200 OK): Updated supplier object or 404 Not Found.
*   **`DELETE /api/suppliers/{supplierId}`**
    *   Logic: Consider implications if supplier is linked to products. May prevent deletion if linked, or set `supplier_id` to NULL on products. For simplicity, prevent deletion if linked.
    *   Response (204 No Content) or (400 Bad Request if linked to products) or 404 Not Found.

### 2. Product CRUD API Endpoints (Admin)

*   **`POST /api/products`**
    *   Request: `{ name, sku, barcode, description, category, supplier_id, purchase_price, retail_price, current_quantity (optional, default 0), reorder_level, is_sellable, is_professional_use_only }`
    *   Logic: If `current_quantity` is provided and > 0, create an `InventoryAdjustments` record with type "initial_stock".
    *   Response (201 Created): New product object.
*   **`GET /api/products`**
    *   Query Params: `search` (name, sku, barcode), `category`, `supplierId`, `isSellable` (boolean), `isProfessionalUseOnly` (boolean), `lowStock` (boolean, to find products at or below `reorder_level`), `page`, `limit`.
    *   Response (200 OK): `{ products: [...], totalPages, currentPage }`.
*   **`GET /api/products/{productId}`**
    *   Response (200 OK): Single product object or 404 Not Found.
*   **`PUT /api/products/{productId}`**
    *   Request: Fields to update from product schema.
    *   Logic:
        *   If `current_quantity` is directly updated here by an Admin, it should ideally be for correcting a count. This action *should* also create an `InventoryAdjustments` record (e.g., type "manual_count", `quantity_change` would be `new_quantity - old_quantity`).
        *   It's generally better to use the dedicated adjustment endpoint for stock changes other than initial setup or full edits.
    *   Response (200 OK): Updated product object or 404 Not Found.
*   **`DELETE /api/products/{productId}`**
    *   Logic: Soft delete (set `is_sellable = false` and add a flag like `is_archived = true`) is preferred if the product has associated transaction history or inventory adjustments. Or, prevent deletion if linked to historical data. For simplicity: prevent deletion if `TransactionItems` or `InventoryAdjustments` exist for it.
    *   Response (204 No Content) or (400 Bad Request if linked) or 404 Not Found.

### 3. Inventory Adjustment API Endpoints (Admin)

*   **`POST /api/products/{productId}/adjustments`**
    *   Permissions: Admin.
    *   Request: `{ "quantity_change": -2, "adjustment_type": "damage", "reason": "Dropped during stocking", "adjusted_by_user_id": X }`
    *   Logic:
        1.  Validate `adjustment_type`.
        2.  Start a database transaction.
        3.  Update `Products.current_quantity` by adding `quantity_change`. Ensure `current_quantity` does not go below zero if business rules enforce this (or allow negative for tracking discrepancies).
        4.  Create a new record in `InventoryAdjustments`.
        5.  Commit database transaction.
    *   Response (201 Created): New inventory adjustment object.
*   **`GET /api/products/{productId}/adjustments`**
    *   Permissions: Admin.
    *   Query Params: `page`, `limit`.
    *   Response (200 OK): `{ adjustments: [...], totalPages, currentPage }`.

### 4. POS Integration (Modify `POST /api/pos/transactions` from `pos_module_design.md`)

*   **Modified Request Body for `TransactionItems` within `POST /api/pos/transactions`:**
    *   An item can now have `productId` OR `serviceId`.
    ```json
    // Existing TransactionItem structure for services:
    // { "serviceId": 10, "staffId": 5, "itemName": "Women's Haircut", "quantity": 1, "unitPrice": 60.00 }
    // New structure for products:
    {
      "productId": 101, // ID of the product from Products table
      "staffId": 5,     // Staff member credited with sale (optional for products, or salon default)
      // "itemName" and "unitPrice" will be fetched from Products table by the backend
      "quantity": 2
    }
    ```
*   **Modified Backend Logic for `POST /api/pos/transactions`:**
    1.  **Item Processing (within the loop for `request.items`):**
        *   If `item.productId` is present:
            *   Fetch the product from `Products` table using `item.productId`.
            *   If product not found or `product.is_sellable == false`, return an error.
            *   Check if `product.current_quantity >= item.quantity`. If not, return an error (insufficient stock).
            *   The `itemName` for `TransactionItems` table should be `product.name`.
            *   The `unitPrice` for `TransactionItems` table should be `product.retail_price`.
        *   Else (if `item.serviceId` is present):
            *   Proceed as per existing logic (fetch service details).
    2.  **After `Transactions` and `TransactionItems` records are successfully created (within the same database transaction):**
        *   For each `TransactionItem` that was a product sale:
            *   Decrement `Products.current_quantity` for the `productId` by `TransactionItem.quantity`.
            *   Create an `InventoryAdjustments` record:
                *   `product_id`: The sold product's ID.
                *   `quantity_change`: `- TransactionItem.quantity` (negative value).
                *   `adjustment_type`: "sale".
                *   `reason`: `Transaction ID: {transaction_id_from_parent_table}`.
                *   `adjusted_by_user_id`: Staff member who processed the POS transaction (if available).
    3.  Ensure all database operations (creating transaction, items, updating product quantities, creating adjustments) are partall of a single database transaction to maintain data integrity.

## III. Frontend Outline (React)

### 1. Supplier Management Components (Admin View - e.g., `/admin/suppliers`)

*   **`SupplierListPage` Component:**
    *   Fetches and displays suppliers in `SupplierTable`.
    *   Search/filter options.
    *   Button to "Add New Supplier" (opens `SupplierFormModal`).
*   **`SupplierTable` Component:**
    *   Table with columns: Name, Contact Person, Email, Phone.
    *   Actions: Edit, Delete.
*   **`SupplierFormModal` Component:**
    *   Form for creating/editing supplier details.
    *   Calls `POST /api/suppliers` or `PUT /api/suppliers/{supplierId}`.

### 2. Product Management Components (Admin View - e.g., `/admin/products`)

*   **`ProductListPage` Component:**
    *   Fetches and displays products in `ProductTable`.
    *   Filters: Category, Supplier, Sellable status, Low Stock. Search by name/SKU.
    *   Button to "Add New Product" (opens `ProductFormModal`).
*   **`ProductTable` Component:**
    *   Table: Name, SKU, Category, Supplier, Purchase Price, Retail Price, Current Quantity, Reorder Level, Sellable.
    *   Actions: Edit, Delete, "Adjust Stock" (opens `InventoryAdjustmentModal`).
*   **`ProductFormModal` Component:**
    *   Form for all product fields (create/edit).
    *   Calls `POST /api/products` or `PUT /api/products/{productId}`.
*   **`InventoryAdjustmentModal` Component (or section within Product Edit):**
    *   Triggered from "Adjust Stock" button on `ProductTable` or product details.
    *   Form fields: `quantity_change` (can be +/-), `adjustment_type` (dropdown: manual count, damage, etc.), `reason`.
    *   Displays current quantity before adjustment.
    *   Calls `POST /api/products/{productId}/adjustments`.
*   **`ProductAdjustmentHistoryPage` (Optional, linked from product details):**
    *   Displays list of `InventoryAdjustments` for a specific product.

### 3. POS Interface Enhancements (Staff/Admin View - in `/pos` or `/checkout`, modifying `POSCheckoutPage` from `pos_module_design.md`)

*   **`TransactionItemCreator` Sub-Component (within `POSCheckoutPage`):**
    *   Add a toggle/tabs to switch between "Add Service" and "Add Product".
    *   **If "Add Product" is selected:**
        *   Input field to search products by name, SKU, or barcode (connects to `GET /api/products?isSellable=true&search=...`).
        *   Displays search results (name, retail price, current quantity).
        *   Selecting a product auto-fills `itemName` (product name) and `unitPrice` (product retail price) for the cart item.
        *   Input for `quantity`.
        *   Warning if selected `quantity` exceeds `current_quantity`.
    *   The "Add to Cart" button adds the product item to the `TransactionCart`.
*   **`TransactionCart` Sub-Component:**
    *   No major changes needed, as it should already be generic enough to display items with name, quantity, unit price, and total price.

## IV. Deliverables Summary

*   **Database Schemas:** SQL for `Suppliers`, `Products`, `InventoryAdjustments`.
*   **API Endpoint Definitions:**
    *   Supplier CRUD APIs: Request/response structures.
    *   Product CRUD APIs: Request/response structures.
    *   Inventory Adjustment APIs: Request/response structures.
*   **POS Integration:**
    *   Detailed modifications to `POST /api/pos/transactions` request structure (for `TransactionItems`) and backend logic to handle product sales, stock deduction, and creation of "sale" type inventory adjustments.
*   **Frontend Component Outlines:**
    *   Admin components for Supplier management (`SupplierListPage`, `SupplierFormModal`).
    *   Admin components for Product management (`ProductListPage`, `ProductFormModal`, `InventoryAdjustmentModal`).
    *   Enhancements to existing POS UI (`TransactionItemCreator`) to allow adding products to transactions.

This design provides a comprehensive plan for the Basic Inventory Management system and its integration with the POS.
