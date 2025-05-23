# Point of Sale (POS) System Enhancements - Design Document

This document outlines the design for enhancements to the Point of Sale (POS) system, including real payment gateway integrations, split payments, tip processing, refunds/exchanges, and discount/promotion application. It references `pos_module_design.md` and `financial_tracking_module_design.md`.

## I. Data Model Design (PostgreSQL)

### 1. `Transactions` Table (Modifications)

*   Remove `payment_method_mock`.
*   Remove `transaction_reference_mock`.
*   Rename `status` to `transaction_status` for clarity.
*   Rename `final_amount` to `grand_total` to avoid confusion after discounts.
*   Rename `discount_amount` to `transaction_level_discount_amount`.
*   **New Fields:**
    *   `payment_gateway` (TEXT, Nullable, e.g., "stripe", "paypal", "srilankan_gw", "cash", "other_manual"): Stores the primary or first gateway if not split, or "multiple" if split. "cash" or "other_manual" for non-gateway payments.
    *   `gateway_transaction_id` (TEXT, Nullable): Primary ID from the payment provider if single payment.
    *   `payment_status` (TEXT NOT NULL DEFAULT 'pending', CHECK (`payment_status` IN ('pending', 'awaiting_capture', 'succeeded', 'failed', 'partially_paid', 'paid_multiple', 'refunded', 'partially_refunded'))).
    *   `tip_amount` (DECIMAL(10, 2) DEFAULT 0.00 NOT NULL CHECK (`tip_amount` >= 0)).
    *   `subtotal_before_discounts` (DECIMAL(10, 2) NOT NULL): Sum of `TransactionItems.total_price` before any discounts.
    *   `total_item_level_discount_amount` (DECIMAL(10, 2) DEFAULT 0.00 NOT NULL): Sum of discounts applied to individual items.
    *   `applied_transaction_promotion_id` (INTEGER, REFERENCES `Promotions`(`id`) ON DELETE SET NULL, Nullable): For transaction-wide promotions.

*Updated `Transactions` Table structure snippet (conceptual):*
```sql
-- Existing fields like id, customer_id, created_at, notes remain.
-- total_amount (from pos_module_design.md) is now subtotal_before_discounts.
-- tax_amount remains.

ALTER TABLE Transactions DROP COLUMN payment_method_mock;
ALTER TABLE Transactions DROP COLUMN transaction_reference_mock;
ALTER TABLE Transactions RENAME COLUMN status TO transaction_status;
ALTER TABLE Transactions RENAME COLUMN final_amount TO grand_total; -- Will be calculated as: subtotal_before_discounts - total_item_level_discount_amount - transaction_level_discount_amount + tax_amount + tip_amount
ALTER TABLE Transactions RENAME COLUMN discount_amount TO transaction_level_discount_amount; -- For discounts applied to the whole transaction

ALTER TABLE Transactions ADD COLUMN payment_gateway TEXT;
ALTER TABLE Transactions ADD COLUMN gateway_transaction_id TEXT; -- Primary gateway ID
ALTER TABLE Transactions ADD COLUMN payment_status TEXT NOT NULL DEFAULT 'pending' CHECK (payment_status IN ('pending', 'awaiting_capture', 'succeeded', 'failed', 'partially_paid', 'paid_multiple', 'refunded', 'partially_refunded'));
ALTER TABLE Transactions ADD COLUMN tip_amount DECIMAL(10, 2) DEFAULT 0.00 NOT NULL CHECK (tip_amount >= 0);
ALTER TABLE Transactions ADD COLUMN subtotal_before_discounts DECIMAL(10, 2) NOT NULL; -- Sum of TransactionItems.total_price
ALTER TABLE Transactions ADD COLUMN total_item_level_discount_amount DECIMAL(10, 2) DEFAULT 0.00 NOT NULL; -- Sum of TransactionItems.discount_amount_item
ALTER TABLE Transactions ADD COLUMN applied_transaction_promotion_id INTEGER REFERENCES Promotions(id) ON DELETE SET NULL;
```

### 2. `TransactionItems` Table (Modifications)

*   **New Fields:**
    *   `applied_item_promotion_id` (INTEGER, REFERENCES `Promotions`(`id`) ON DELETE SET NULL, Nullable): For item-specific promotions.
    *   `discount_amount_item` (DECIMAL(10, 2) DEFAULT 0.00 NOT NULL): Discount applied specifically to this item.
    *   `price_after_discount` (DECIMAL(10, 2) NOT NULL): Calculated as `total_price - discount_amount_item`. This is the value that contributes to the transaction's financial sum.

*Updated `TransactionItems` Table structure snippet (conceptual):*
```sql
-- Existing fields like id, transaction_id, service_id, product_id, item_name, quantity, unit_price, total_price (quantity * unit_price) remain.
ALTER TABLE TransactionItems ADD COLUMN applied_item_promotion_id INTEGER REFERENCES Promotions(id) ON DELETE SET NULL;
ALTER TABLE TransactionItems ADD COLUMN discount_amount_item DECIMAL(10, 2) DEFAULT 0.00 NOT NULL CHECK (discount_amount_item >= 0 AND discount_amount_item <= total_price);
ALTER TABLE TransactionItems ADD COLUMN price_after_discount DECIMAL(10, 2) NOT NULL; -- Calculated: total_price - discount_amount_item
```
*(Note: `total_price` remains `quantity * unit_price` before item-specific discount. The sum of `price_after_discount` for all items will be the new `subtotal_before_discounts` for the transaction, before transaction-level discounts are applied).*

### 3. `TransactionCommissions` Table (New)

*   For Sri Lankan gateway commission.
*   **Schema:**
    *   `id` (SERIAL PRIMARY KEY)
    *   `transaction_id` (INTEGER NOT NULL REFERENCES `Transactions`(`id`) ON DELETE CASCADE)
    *   `payment_id` (INTEGER, REFERENCES `TransactionPayments`(`id`) ON DELETE CASCADE, Nullable) -- If commission is per payment part
    *   `commission_rate` (DECIMAL(5, 4) NOT NULL) -- e.g., 0.0500 for 5%
    *   `commission_amount` (DECIMAL(10, 2) NOT NULL)
    *   `calculated_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)
    *   UNIQUE (`transaction_id`, `payment_id`) -- Ensure commission is logged once per payment if applicable

*SQL for `TransactionCommissions` Table:*
```sql
CREATE TABLE TransactionCommissions (
    id SERIAL PRIMARY KEY,
    transaction_id INTEGER NOT NULL REFERENCES Transactions(id) ON DELETE CASCADE,
    payment_id INTEGER REFERENCES TransactionPayments(id) ON DELETE CASCADE, -- Link to specific payment if commission is per payment part
    commission_rate DECIMAL(5, 4) NOT NULL,
    commission_amount DECIMAL(10, 2) NOT NULL,
    calculated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
    UNIQUE (transaction_id, payment_id)
);
```

### 4. `TransactionPayments` Table (New)

*   For split payments.
*   **Schema:**
    *   `id` (SERIAL PRIMARY KEY)
    *   `transaction_id` (INTEGER NOT NULL REFERENCES `Transactions`(`id`) ON DELETE CASCADE)
    *   `amount_paid` (DECIMAL(10, 2) NOT NULL CHECK (`amount_paid` > 0))
    *   `payment_gateway` (TEXT NOT NULL, e.g., "stripe", "paypal", "srilankan_gw", "cash", "other_manual")
    *   `gateway_transaction_id` (TEXT, Nullable) -- ID from the payment provider for this specific payment part
    *   `payment_method_details` (TEXT, Nullable) -- E.g., "Visa ****4242", "Cash", "PayPal user@example.com"
    *   `payment_status` (TEXT NOT NULL DEFAULT 'succeeded', CHECK (`payment_status` IN ('pending', 'succeeded', 'failed', 'refunded'))) -- Status of this specific payment part
    *   `paid_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)

*SQL for `TransactionPayments` Table:*
```sql
CREATE TABLE TransactionPayments (
    id SERIAL PRIMARY KEY,
    transaction_id INTEGER NOT NULL REFERENCES Transactions(id) ON DELETE CASCADE,
    amount_paid DECIMAL(10, 2) NOT NULL CHECK (amount_paid > 0),
    payment_gateway TEXT NOT NULL,
    gateway_transaction_id TEXT,
    payment_method_details TEXT,
    payment_status TEXT NOT NULL DEFAULT 'succeeded' CHECK (payment_status IN ('pending', 'succeeded', 'failed', 'refunded')),
    paid_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL
);
```

### 5. `Promotions` Table (New)

*   **Schema:**
    *   `id` (SERIAL PRIMARY KEY)
    *   `name` (TEXT NOT NULL)
    *   `description` (TEXT)
    *   `promo_code` (TEXT UNIQUE, Nullable) -- Can be null if promotion is auto-applied
    *   `discount_type` (TEXT NOT NULL, CHECK (`discount_type` IN ('percentage', 'fixed_amount')))
    *   `discount_value` (DECIMAL(10, 2) NOT NULL CHECK (`discount_value` > 0))
    *   `applicable_scope` (TEXT NOT NULL, CHECK (`applicable_scope` IN ('all_services', 'specific_services', 'all_products', 'specific_products', 'entire_transaction')))
    *   `applicable_ids` (INTEGER[], Nullable) -- Array of Service IDs or Product IDs if scope is specific
    *   `minimum_purchase_amount` (DECIMAL(10,2) DEFAULT 0.00) -- Minimum cart subtotal (after item discounts) for transaction-wide promos
    *   `start_date` (TIMESTAMP WITH TIME ZONE NOT NULL)
    *   `end_date` (TIMESTAMP WITH TIME ZONE NOT NULL)
    *   `is_active` (BOOLEAN DEFAULT TRUE NOT NULL)
    *   `max_uses` (INTEGER, Nullable) -- Max total uses for this promotion
    *   `current_uses` (INTEGER DEFAULT 0)
    *   `max_uses_per_customer` (INTEGER, Nullable)
    *   `created_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)
    *   `updated_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)
    *   CONSTRAINT promo_end_date_check CHECK (end_date > start_date)

*SQL for `Promotions` Table:*
```sql
CREATE TABLE Promotions (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT,
    promo_code TEXT UNIQUE,
    discount_type TEXT NOT NULL CHECK (discount_type IN ('percentage', 'fixed_amount')),
    discount_value DECIMAL(10, 2) NOT NULL CHECK (discount_value > 0),
    applicable_scope TEXT NOT NULL CHECK (applicable_scope IN ('all_services', 'specific_services', 'all_products', 'specific_products', 'entire_transaction')),
    applicable_ids INTEGER[],
    minimum_purchase_amount DECIMAL(10,2) DEFAULT 0.00,
    start_date TIMESTAMP WITH TIME ZONE NOT NULL,
    end_date TIMESTAMP WITH TIME ZONE NOT NULL,
    is_active BOOLEAN DEFAULT TRUE NOT NULL,
    max_uses INTEGER,
    current_uses INTEGER DEFAULT 0,
    max_uses_per_customer INTEGER,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
    CONSTRAINT promo_end_date_check CHECK (end_date > start_date)
);
CREATE INDEX idx_promotions_code ON Promotions(promo_code) WHERE promo_code IS NOT NULL;
CREATE INDEX idx_promotions_active_dates ON Promotions(is_active, start_date, end_date);
```

## II. Payment Gateway Integration

### 1. Backend Setup (General)

*   **Node.js SDKs:** Use official SDKs (e.g., `stripe`, `paypal-rest-sdk` or `@paypal/checkout-server-sdk`). For Sri Lankan gateway, use provided SDK or HTTP client for direct API calls.
*   **Secure Configuration:** API keys/secrets stored in environment variables, loaded via config service.
*   **`PaymentService` Abstraction (Conceptual):**
    ```typescript
    interface PaymentIntentResult { clientSecret?: string; nextActionUrl?: string; paymentId?: string; }
    interface PaymentProcessResult { success: boolean; transactionId?: string; message?: string; }
    interface RefundResult { success: boolean; refundId?: string; message?: string; }

    interface IPaymentGateway {
      createPaymentIntent(transactionId: number, amount: number, currency: string, paymentMethodId?: string, customerGatewayId?: string, metadata?: object): Promise<PaymentIntentResult>;
      processPayment(paymentIntentId: string, metadata?: object): Promise<PaymentProcessResult>; // For gateways that need a separate confirm step
      handleWebhook(payload: any, signature: string): Promise<boolean>; // Returns true if successfully processed
      refundPayment(gatewayTransactionId: string, amount: number, currency: string, reason?: string, metadata?: object): Promise<RefundResult>;
    }
    // Implement StripePaymentGateway, PayPalPaymentGateway, SriLankanPaymentGateway
    ```
    A factory or strategy pattern would select the appropriate gateway implementation.

### 2. API Endpoints (Backend)

*   **`POST /api/pos/transactions` (Modified - from `pos_module_design.md`)**
    *   Logic:
        1.  Calculate `subtotal_before_discounts` (sum of `TransactionItems.total_price`).
        2.  Apply item-level promotions (update `TransactionItems.discount_amount_item`, `TransactionItems.price_after_discount`, `TransactionItems.applied_item_promotion_id`).
        3.  Calculate `total_item_level_discount_amount`.
        4.  Apply transaction-level promotion (update `Transactions.transaction_level_discount_amount`, `Transactions.applied_transaction_promotion_id`).
        5.  Calculate `grand_total = (sum of TransactionItems.price_after_discount) - Transactions.transaction_level_discount_amount + Transactions.tax_amount + Transactions.tip_amount`. (Tip is added later at payment stage).
        6.  Create `Transactions` record with `payment_status = 'pending'`.
        7.  Create `TransactionItems` records.
        8.  Return transaction object including `grand_total` and `id`.
    *   The actual payment happens in a subsequent step.

*   **`POST /api/pos/transactions/{transactionId}/initiate-payment-part` (New - for Payment Parts)**
    *   Permissions: Staff, Admin.
    *   Request Body:
        ```json
        {
          "amountToPay": 50.00, // Amount for this specific payment part
          "paymentGateway": "stripe", // "paypal", "srilankan_gw", "cash"
          "paymentMethodId": "pm_...", // For Stripe, from frontend Stripe.js/Elements
          "tipAmountForThisPayment": 5.00, // Optional, if tip is split or attributed to one payment part
          "customerGatewayId": "cus_..." // Optional, Stripe customer ID
        }
        ```
    *   Logic:
        1.  Fetch `Transactions` record by `transactionId`. Verify it's still 'pending' or 'partially_paid'.
        2.  Calculate total amount for this payment part: `amountToPay + tipAmountForThisPayment`.
        3.  Use `PaymentService` for the chosen `paymentGateway` to create payment intent/charge.
        4.  If "cash" or "other_manual":
            *   Record `TransactionPayments` directly with status 'succeeded'.
            *   Update `Transactions.payment_status` (e.g., to 'partially_paid' or 'succeeded').
            *   Update `Transactions.tip_amount` (summing tips from all parts).
            *   Return success.
        5.  For gateways like Stripe: Return `clientSecret` or other details needed by frontend to complete payment. The actual recording of `TransactionPayments` and `Transactions` update happens via webhook or a confirmation step.
    *   Response: `{ paymentPartId (temp), clientSecret, nextActionUrl, status }`

*   **`POST /api/pos/transactions/{transactionId}/confirm-payment-part/{paymentPartId}` (New - Optional)**
    *   Sometimes needed if frontend confirmation is required after 3DS or redirect.
    *   Logic: Confirms a payment part, records `TransactionPayments`, updates `Transactions.payment_status`, `Transactions.tip_amount`.

*   **`POST /api/webhooks/{gatewayName}` (e.g., `/api/webhooks/stripe`, `/api/webhooks/paypal`)**
    *   Public, but signature verification is CRITICAL.
    *   Logic:
        1.  Verify webhook signature using gateway-specific secret.
        2.  Parse payload to get payment status, amount, `gateway_transaction_id`, and associated `transactionId` (often stored in metadata during intent creation).
        3.  If payment succeeded:
            *   Create/Update `TransactionPayments` record for this part.
            *   Update `Transactions.payment_status` (if all parts paid, to 'succeeded').
            *   Update `Transactions.gateway_transaction_id` (if it's the primary/only payment).
            *   Update `Transactions.tip_amount` (if tip was part of this payment).
            *   If Sri Lankan Gateway: Calculate 5% commission on `TransactionPayments.amount_paid` (excluding tip), store in `TransactionCommissions`.
            *   (Inventory deduction for products was moved to when transaction is fully paid or items are taken - see V. Refunds/Exchanges)
        4.  If payment failed: Update `TransactionPayments` and `Transactions.payment_status`.
    *   Response: HTTP 200 OK to acknowledge receipt to gateway.

### 3. Specific Gateway Considerations

*   **Stripe/PayPal:** Use official SDKs. Stripe Elements or PayPal JS SDK for frontend card input.
*   **Sri Lankan Gateway:**
    *   Research API: Direct HTTP calls if no SDK. Authentication, payment initiation, redirect handling.
    *   Commission Logic: Implement as described in webhook. `commission_amount = TransactionPayments.amount_paid * 0.05`.

## III. Split Payments

*   **Data Model:** `TransactionPayments` table handles this.
*   **API Logic:**
    *   `POST /api/pos/transactions/{transactionId}/initiate-payment-part` is called multiple times by frontend for different payment methods/amounts until `Transactions.grand_total` (plus total tip) is covered.
    *   Backend needs to track amount paid so far against `grand_total + total_tip_amount`.
*   **UI (Frontend):**
    *   Display "Amount Due".
    *   Allow adding multiple payment methods:
        *   "Pay $X with Card", "Pay $Y with Cash".
        *   Each triggers `initiate-payment-part` for the specified amount and method.

## IV. Tip Processing

*   **Data Model:** `Transactions.tip_amount` (total tip). `TransactionPayments` can optionally store `tip_amount_for_this_payment` if needed for reconciliation, or tip is summed up into `Transactions.tip_amount` as payments are confirmed.
*   **API Logic:**
    *   `initiate-payment-part` request takes `tipAmountForThisPayment`. This is added to the amount charged by the gateway for that payment part.
    *   Total tip is aggregated in `Transactions.tip_amount`.
*   **UI (Frontend):**
    *   Option to add tip (percentage or fixed amount) before/during payment part processing.
    *   If multiple staff involved in items, UI might allow splitting tip (advanced, not for MVP).

## V. Refunds/Exchanges

### 1. Refunds API (Backend)

*   **`POST /api/pos/transactions/{transactionId}/refund`**
    *   Permissions: Admin, or Staff with specific permissions.
    *   Request Body:
        ```json
        {
          "paymentIdToRefund": 123, // ID from TransactionPayments if refunding a specific payment part
          "amount": 20.00, // Amount to refund
          "reason": "Customer dissatisfaction",
          "itemsToRestock": [ // Optional
            { "transactionItemId": 1, "productId": 101, "quantity": 1 }
          ]
        }
        ```
    *   Logic:
        1.  Fetch `Transaction` and relevant `TransactionPayments` record.
        2.  Use `PaymentService` to process refund via the original gateway for that `paymentIdToRefund`, using its `gateway_transaction_id`.
        3.  If refund successful:
            *   Update `TransactionPayments.payment_status` to "refunded".
            *   Update `Transactions.payment_status` (e.g., "partially_refunded", "refunded").
            *   Update `Transactions.grand_total` and potentially other amount fields to reflect the refund.
            *   If `itemsToRestock`:
                *   For each item, fetch `Product` by `productId`.
                *   Increase `Products.current_quantity` by `quantity`.
                *   Create `InventoryAdjustments` record (type "customer_return", `quantity_change` is positive).
        4.  (Inventory deduction logic at point of sale): Inventory for products should be deducted when the transaction `payment_status` becomes 'succeeded' or when items are physically handed over (if policy allows payment later). If payment is 'pending' but items given, stock must be deducted. For simplicity, deduct stock when `Transactions.payment_status` becomes 'succeeded' or 'paid_multiple'.
    *   Response (200 OK): Updated transaction object or success message.

### 2. Exchanges Logic

*   Handle as a refund of original item(s) and a new sale (separate transaction) of new item(s).
*   UI can streamline this to appear as one flow for the user, but backend processes two distinct operations.

### 3. UI (Frontend)

*   **Transaction History View:**
    *   "Refund" button on eligible transactions/transaction items.
    *   **Refund Form:**
        *   Select payment part to refund (if multiple).
        *   Input refund `amount` (cannot exceed original payment part amount).
        *   Input `reason`.
        *   Checklist/input for `itemsToRestock` (pre-filled from original transaction, quantity editable up to original item quantity).

## VI. Discount & Promotion Application

### 1. Data Model

*   `Promotions` table defined.
*   `TransactionItems.applied_item_promotion_id`, `TransactionItems.discount_amount_item`.
*   `Transactions.applied_transaction_promotion_id`, `Transactions.transaction_level_discount_amount`.
*   `Transactions.total_item_level_discount_amount`.

### 2. API Logic

*   **`POST /api/pos/transactions` (Modified as described in II.2):**
    *   Takes `promo_code` or `promotion_id` (array for multiple item/transaction promos).
    *   Backend fetches promotion details.
    *   Validates applicability (dates, scope, IDs, `max_uses`, `max_uses_per_customer` - customer ID needed for this check).
    *   Calculates discounts:
        *   For item-specific promos: update `TransactionItems.discount_amount_item` and `price_after_discount`.
        *   For transaction-wide promos: update `Transactions.transaction_level_discount_amount`.
    *   Updates `Promotions.current_uses`.
    *   Updates `Transactions.subtotal_before_discounts`, `Transactions.total_item_level_discount_amount`, and `Transactions.grand_total`.
*   **`GET /api/promotions/validate` (New - or part of a general "calculate cart" endpoint)**
    *   Permissions: Staff, Admin, Customer (if used in online booking).
    *   Request Query Params: `?code={promo_code}&customerId={optional}&contextItems=[{productId/serviceId, quantity, unitPrice}, ...]`
    *   Response: `{ isValid: true/false, promotionDetails: {...}, applicableDiscount: X.XX, error: "..." }`

### 3. UI (Frontend)

*   **POS Checkout Screen:**
    *   Input field for "Promo Code".
    *   On apply, calls `GET /api/promotions/validate` (or backend recalculates entire transaction with promo).
    *   Display applied discounts per item and on transaction total.
    *   Show updated `grand_total`.
*   **Admin UI for Promotions (e.g., `/admin/promotions`):**
    *   `PromotionListPage`: List, filter, search promotions.
    *   `PromotionFormPage`: CRUD operations for `Promotions` table.

## VII. Deliverables Summary

*   **Database Schema Changes/Additions:**
    *   Modified `Transactions` (payment fields, status, discount/tip totals).
    *   Modified `TransactionItems` (promo/discount fields).
    *   New `TransactionCommissions` table.
    *   New `TransactionPayments` table (for split payments).
    *   New `Promotions` table.
*   **API Designs:**
    *   Payment Initiation: `POST /api/pos/transactions/{transactionId}/initiate-payment-part`.
    *   Webhooks: `POST /api/webhooks/{gatewayName}`.
    *   Refunds: `POST /api/pos/transactions/{transactionId}/refund`.
    *   Promotion Validation: `GET /api/promotions/validate`.
    *   Modifications to `POST /api/pos/transactions` for new calculation flow.
*   **Logic Outlines:**
    *   `PaymentService` abstraction.
    *   Sri Lankan gateway commission calculation.
    *   Discount application logic.
    *   Inventory deduction point (on successful payment).
*   **UI Outlines:**
    *   POS: Payment method selection, tip entry, split payment handling, refund form, promo code input.
    *   Admin: Promotion management CRUD.

This design provides a comprehensive plan for these POS enhancements.
