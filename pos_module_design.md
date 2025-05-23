# Point of Sale (POS) Module - API, Database, and Frontend Design

This document outlines the design for a Basic Point of Sale (POS) system, focusing on recording transactions, generating digital receipts, and mock payment processing. It references existing designs for Services, Appointments, and Customers.

## I. Data Model Design (PostgreSQL)

1.  **`Transactions` Table:**
    *   Stores overall information for each transaction.
    *   **Schema:**
        *   `id` (SERIAL PRIMARY KEY)
        *   `customer_id` (INTEGER, REFERENCES `Users`(`id`) ON DELETE SET NULL, Nullable): Links to the customer if they are registered and selected. `ON DELETE SET NULL` preserves transaction history if a customer account is deleted.
        *   `created_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)
        *   `total_amount` (DECIMAL(10, 2) NOT NULL): Sum of `TransactionItems.total_price` before discounts and taxes.
        *   `tax_amount` (DECIMAL(10, 2) DEFAULT 0.00 NOT NULL): Placeholder for tax calculations.
        *   `discount_amount` (DECIMAL(10, 2) DEFAULT 0.00 NOT NULL): Placeholder for discounts.
        *   `final_amount` (DECIMAL(10, 2) NOT NULL): Calculated as `total_amount + tax_amount - discount_amount`.
        *   `payment_method_mock` (TEXT NOT NULL, CHECK (`payment_method_mock` IN ('cash_mock', 'card_mock', 'other_mock'))): Mocked payment method used.
        *   `status` (TEXT NOT NULL DEFAULT 'completed', CHECK (`status` IN ('completed', 'pending_payment', 'cancelled', 'refunded'))): Status of the transaction.
        *   `notes` (TEXT, Nullable): Optional notes for the transaction.
        *   `transaction_reference_mock` (TEXT, Nullable): Placeholder for a mock payment processor reference ID.

    *SQL for `Transactions` Table:*
    ```sql
    CREATE TABLE Transactions (
        id SERIAL PRIMARY KEY,
        customer_id INTEGER REFERENCES Users(id) ON DELETE SET NULL,
        created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
        total_amount DECIMAL(10, 2) NOT NULL CHECK (total_amount >= 0),
        tax_amount DECIMAL(10, 2) DEFAULT 0.00 NOT NULL CHECK (tax_amount >= 0),
        discount_amount DECIMAL(10, 2) DEFAULT 0.00 NOT NULL CHECK (discount_amount >= 0),
        final_amount DECIMAL(10, 2) NOT NULL CHECK (final_amount >= 0),
        payment_method_mock TEXT NOT NULL CHECK (payment_method_mock IN ('cash_mock', 'card_mock', 'other_mock')),
        status TEXT NOT NULL DEFAULT 'completed' CHECK (status IN ('completed', 'pending_payment', 'cancelled', 'refunded')),
        notes TEXT,
        transaction_reference_mock TEXT,
        CONSTRAINT fk_customer FOREIGN KEY (customer_id) REFERENCES Users(id) ON DELETE SET NULL
    );
    CREATE INDEX idx_transactions_customer_id ON Transactions(customer_id);
    CREATE INDEX idx_transactions_created_at ON Transactions(created_at);
    CREATE INDEX idx_transactions_status ON Transactions(status);
    ```

2.  **`TransactionItems` Table:**
    *   Stores individual items (services or custom products) within each transaction.
    *   **Schema:**
        *   `id` (SERIAL PRIMARY KEY)
        *   `transaction_id` (INTEGER NOT NULL, REFERENCES `Transactions`(`id`) ON DELETE CASCADE): Links to the parent transaction. `ON DELETE CASCADE` ensures items are removed if the transaction is deleted (though transactions are usually soft-deleted or marked cancelled/refunded).
        *   `service_id` (INTEGER, REFERENCES `Services`(`id`) ON DELETE SET NULL, Nullable): Links to a service from the catalog if the item is a predefined service. `ON DELETE SET NULL` preserves the transaction item record even if the service is later deleted from the catalog (though services are also typically soft-deleted).
        *   `staff_id` (INTEGER, REFERENCES `Users`(`id`) ON DELETE SET NULL, Nullable): Staff member who performed or is credited for the service/sale. `ON DELETE SET NULL` if staff account is deleted.
        *   `appointment_id` (INTEGER, REFERENCES `Appointments`(`id`) ON DELETE SET NULL, Nullable): Links to an appointment if the transaction item originated from one. `ON DELETE SET NULL` if appointment is deleted.
        *   `item_name` (TEXT NOT NULL): Name of the item (e.g., service name, custom product name). This is crucial for historical accuracy if the linked service name changes later.
        *   `quantity` (INTEGER NOT NULL DEFAULT 1 CHECK (`quantity` > 0))
        *   `unit_price` (DECIMAL(10, 2) NOT NULL CHECK (`unit_price` >= 0)): Price per unit at the time of transaction. This is also for historical accuracy.
        *   `total_price` (DECIMAL(10, 2) NOT NULL CHECK (`total_price` >= 0)): Calculated as `quantity * unit_price`.

    *SQL for `TransactionItems` Table:*
    ```sql
    CREATE TABLE TransactionItems (
        id SERIAL PRIMARY KEY,
        transaction_id INTEGER NOT NULL REFERENCES Transactions(id) ON DELETE CASCADE,
        service_id INTEGER REFERENCES Services(id) ON DELETE SET NULL,
        staff_id INTEGER REFERENCES Users(id) ON DELETE SET NULL, -- Assuming staff are Users with 'staff' role
        appointment_id INTEGER REFERENCES Appointments(id) ON DELETE SET NULL,
        item_name TEXT NOT NULL,
        quantity INTEGER NOT NULL DEFAULT 1 CHECK (quantity > 0),
        unit_price DECIMAL(10, 2) NOT NULL CHECK (unit_price >= 0),
        total_price DECIMAL(10, 2) NOT NULL CHECK (total_price >= 0), -- quantity * unit_price
        CONSTRAINT fk_transaction FOREIGN KEY (transaction_id) REFERENCES Transactions(id) ON DELETE CASCADE,
        CONSTRAINT fk_service FOREIGN KEY (service_id) REFERENCES Services(id) ON DELETE SET NULL,
        CONSTRAINT fk_staff FOREIGN KEY (staff_id) REFERENCES Users(id) ON DELETE SET NULL,
        CONSTRAINT fk_appointment FOREIGN KEY (appointment_id) REFERENCES Appointments(id) ON DELETE SET NULL
    );
    CREATE INDEX idx_transactionitems_transaction_id ON TransactionItems(transaction_id);
    CREATE INDEX idx_transactionitems_service_id ON TransactionItems(service_id);
    CREATE INDEX idx_transactionitems_staff_id ON TransactionItems(staff_id);
    CREATE INDEX idx_transactionitems_appointment_id ON TransactionItems(appointment_id);
    ```

## II. Backend API Development (Node.js/Express & PostgreSQL)

Protected by authentication (`ensureAuthenticated`) and authorization (`ensureRole`) middleware.

1.  **`POST /api/pos/transactions` (Create Transaction)**
    *   Permissions: Staff, Admin.
    *   Request Body:
        ```json
        {
          "customerId": 123, // Optional: User ID of the customer
          "items": [
            {
              "serviceId": 10, // Optional: Service ID from catalog
              "appointmentId": 55, // Optional: Appointment ID
              "staffId": 5, // User ID of staff member
              "itemName": "Women's Haircut", // Pre-filled from serviceId or custom
              "quantity": 1,
              "unitPrice": 60.00 // Pre-filled from serviceId or custom
            },
            { // Example of a custom item not in the service catalog
              "staffId": 5,
              "itemName": "Retail Product X",
              "quantity": 2,
              "unitPrice": 25.00
            }
          ],
          "paymentMethodMock": "card_mock", // "cash_mock", "other_mock"
          "discountAmount": 0.00, // Optional
          "taxAmount": 0.00, // Optional, for now
          "notes": "Customer requested a silent appointment.", // Optional
          "status": "completed" // Optional, defaults to "completed"
        }
        ```
    *   Logic:
        1.  Validate inputs. Ensure `items` array is present and has valid entries.
        2.  For each item, calculate `total_price = quantity * unitPrice`.
        3.  Calculate `total_amount` for the transaction (sum of all `item.total_price`).
        4.  Calculate `final_amount = total_amount + (request.taxAmount || 0) - (request.discountAmount || 0)`.
        5.  Start a database transaction.
        6.  Create a record in the `Transactions` table.
        7.  For each item in the request, create a corresponding record in the `TransactionItems` table, linking to the new transaction ID.
        8.  If an `appointmentId` is provided for an item, consider updating the linked `Appointments.status` to 'Completed' or a similar status indicating it has been processed through POS. This logic should be idempotent.
        9.  Commit the database transaction.
        10. Return the newly created transaction object, including its items.
    *   Response (201 Created):
        ```json
        {
          "id": 1,
          "customerId": 123,
          "createdAt": "YYYY-MM-DDTHH:mm:ss.sssZ",
          "totalAmount": "85.00", // Example: 60 + 25
          "taxAmount": "0.00",
          "discountAmount": "0.00",
          "finalAmount": "85.00",
          "paymentMethodMock": "card_mock",
          "status": "completed",
          "notes": "Customer requested a silent appointment.",
          "transactionReferenceMock": "mock_ref_123xyz", // Generated if applicable
          "items": [
            {
              "id": 1,
              "transactionId": 1,
              "serviceId": 10,
              "appointmentId": 55,
              "staffId": 5,
              "itemName": "Women's Haircut",
              "quantity": 1,
              "unitPrice": "60.00",
              "totalPrice": "60.00"
            },
            {
              "id": 2,
              "transactionId": 1,
              "staffId": 5,
              "itemName": "Retail Product X",
              "quantity": 1, // Corrected from request example, assuming 1 unit of "Retail Product X" at 25.00
              "unitPrice": "25.00",
              "totalPrice": "25.00"
            }
          ]
        }
        ```
        *(Note: The example response for items was adjusted to reflect a possible interpretation of the request - if "Retail Product X" had quantity 2, its totalPrice would be 50.00 and the transaction totalAmount would be 110.00)*

2.  **`GET /api/pos/transactions` (List Transactions)**
    *   Permissions: Admin (full view), Staff (possibly limited to their transactions or recent ones, or based on salon policy).
    *   Query Parameters:
        *   `customerId` (integer)
        *   `staffId` (integer) - Filter by staff involved in `TransactionItems`.
        *   `startDate` (YYYY-MM-DD)
        *   `endDate` (YYYY-MM-DD)
        *   `status` (string)
        *   `page` (integer), `limit` (integer) for pagination.
    *   Response (200 OK):
        ```json
        {
          "transactions": [ /* array of transaction objects, similar to POST response but items might be summary or excluded */ ],
          "totalPages": 10,
          "currentPage": 1
        }
        ```

3.  **`GET /api/pos/transactions/{transactionId}` (Get Single Transaction)**
    *   Permissions: Admin; Staff (if they were involved or have salon-wide read access); Customer (only if `customerId` matches their authenticated ID).
    *   Response (200 OK): Single transaction object with full details including `items` (same structure as `POST` response), or 404 Not Found.

4.  **Receipt Generation (Conceptual)**
    *   **`GET /api/pos/transactions/{transactionId}/receipt`**
        *   Permissions: Same as `GET /api/pos/transactions/{transactionId}`.
        *   Logic: Fetches the full transaction details (transaction record + all transaction items).
        *   Response (200 OK):
            *   For this iteration, the response will be the structured JSON of the transaction, identical to `GET /api/pos/transactions/{transactionId}`. The frontend will be responsible for formatting this JSON into a human-readable receipt.
            ```json
            // Identical to the response of GET /api/pos/transactions/{transactionId}
            {
              "id": 1,
              "customer": { "id": 123, "firstName": "John", "lastName": "Doe", "email": "john@example.com" }, // Enriched customer data
              "staffPerformingServices": [ // Could be a list of distinct staff from items
                  { "id": 5, "firstName": "Alice", "lastName": "Smith" }
              ],
              "salonDetails": { "name": "Glamour Salon", "address": "123 Main St", "phone": "555-1234" }, // Added for receipt
              "createdAt": "YYYY-MM-DDTHH:mm:ss.sssZ",
              "totalAmount": "85.00",
              "taxAmount": "0.00",
              "discountAmount": "0.00",
              "finalAmount": "85.00",
              "paymentMethodMock": "card_mock",
              "status": "completed",
              "notes": "Customer requested a silent appointment.",
              "items": [
                {
                  "itemName": "Women's Haircut",
                  "quantity": 1,
                  "unitPrice": "60.00",
                  "totalPrice": "60.00",
                  "staffName": "Alice Smith" // Enriched item
                },
                {
                  "itemName": "Retail Product X",
                  "quantity": 1,
                  "unitPrice": "25.00",
                  "totalPrice": "25.00",
                  "staffName": "Alice Smith" // Enriched item
                }
              ]
            }
            ```
        *   **Future Enhancements:** Could generate HTML, PDF, or trigger an email with the receipt. For HTML, the backend could render a template.

## III. Frontend Outline (React)

### 1. POS Interface Components (for Staff/Admin View - e.g., in `/pos` or `/checkout`)

*   **`POSCheckoutPage` Component (Main POS View):**
    *   Manages overall state of the current transaction.
    *   **`CustomerSelector` Sub-Component:**
        *   Input to search for existing customers (from `GET /api/customers`).
        *   Option to quickly add a new customer (triggering a modal or simplified form that calls `POST /api/customers`).
        *   Option for "Guest" or anonymous transaction (sets `customerId` to null).
    *   **`TransactionItemCreator` Sub-Component:**
        *   **Service Selection:** Dropdown/search to add services from catalog (`GET /api/services?is_active=true`). Auto-fills `itemName` and `unitPrice`.
        *   **Custom Item Entry:** Fields for `itemName`, `unitPrice`, `quantity` for items not in the catalog.
        *   **Staff Assignment:** Dropdown to select staff member (`GET /api/staff?is_active=true`) for each item.
        *   "Add Item" button to add to the cart.
    *   **`TransactionCart` Sub-Component:**
        *   Displays current items: `itemName`, `quantity`, `unitPrice`, `totalPrice` for each.
        *   Ability to remove items or edit quantity.
        *   Shows `subtotal` (total_amount), `discount` input (if any), `tax` (placeholder), and `final_amount`.
    *   **`PaymentSection` Sub-Component:**
        *   Buttons for mock payment methods: "Pay with Cash", "Pay with Card". Sets `paymentMethodMock`.
        *   Input for `notes`.
    *   **"Complete Transaction" Button:**
        *   Enabled when items are in cart and payment method selected.
        *   Calls `POST /api/pos/transactions` with the constructed payload.
        *   On success, navigates to `TransactionReceiptPage` or displays receipt modal. Handles errors.

*   **`TransactionReceiptPage` Component (or Modal):**
    *   Route: e.g., `/pos/receipt/{transactionId}`
    *   Receives `transactionId` as prop/param.
    *   Fetches transaction details using `GET /api/pos/transactions/{transactionId}/receipt`.
    *   **`ReceiptDisplay` Sub-Component:**
        *   Formats the JSON data into a clean, readable receipt layout.
        *   Includes salon name, date/time, transaction ID, items, prices, totals, payment method.
        *   Customer details if available.
    *   **Action Buttons:**
        *   "Print Receipt": Uses browser's print functionality (`window.print()`) on the formatted receipt.
        *   "Email Receipt" (Placeholder): Would eventually call a backend endpoint like `POST /api/pos/transactions/{transactionId}/email-receipt`.
        *   "New Transaction" button to go back to `POSCheckoutPage`.

*   **`TransactionHistoryPage` Component:**
    *   Route: e.g., `/admin/transactions` or `/pos/history`
    *   Fetches transactions using `GET /api/pos/transactions`.
    *   **`TransactionFilters` Sub-Component:** Inputs for date range, customer search, staff search.
    *   **`TransactionTable` Sub-Component:**
        *   Displays transactions: ID, Date, Customer, Staff, Final Amount, Status.
        *   Each row is clickable to navigate to `TransactionReceiptPage` for that transaction.
    *   Pagination controls.

## IV. Digital Receipt Data Structure

As defined in `GET /api/pos/transactions/{transactionId}/receipt`, the receipt data will be a JSON object containing:
*   Transaction details (ID, timestamp, amounts, payment method, status, notes).
*   Enriched customer details (if any).
*   Enriched staff details (names of staff involved).
*   Basic salon details (name, address, phone - could be hardcoded or from a config for now).
*   A list of items, each with `itemName`, `quantity`, `unitPrice`, `totalPrice`, and `staffName` (who sold/performed).

The frontend `ReceiptDisplay` component will parse this JSON to render a user-friendly format.

## V. Deliverables Summary for this Document (`pos_module_design.md`)

*   **Database Schema:**
    *   `Transactions` table: For overall transaction data.
    *   `TransactionItems` table: For items within each transaction.
    *   SQL definitions and relationships provided.
*   **Backend API Endpoints:**
    *   `POST /api/pos/transactions` (create)
    *   `GET /api/pos/transactions` (list with filters)
    *   `GET /api/pos/transactions/{transactionId}` (get single)
    *   `GET /api/pos/transactions/{transactionId}/receipt` (for receipt data, initially JSON).
    *   Detailed request/response structures, logic, and permissions.
*   **Frontend Component Outline:**
    *   `POSCheckoutPage` with sub-components for customer selection, item creation, cart, and payment.
    *   `TransactionReceiptPage` (or modal) for displaying and printing formatted receipts.
    *   `TransactionHistoryPage` for viewing and filtering past transactions.
*   **Receipt Generation Approach:** Backend provides structured JSON; frontend formats for display/print.

This design provides a comprehensive plan for the Basic POS system.
