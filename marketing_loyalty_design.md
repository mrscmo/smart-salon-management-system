# Basic Marketing & Loyalty Programs - Design Document

This document outlines the design for a points-based loyalty program and basic email marketing integration (e.g., Mailchimp). It references `crm_module_design.md`, `pos_enhancements_design.md`, and `project_setup_plan.md`.

## I. Points-Based Loyalty Program

### 1. Data Model (PostgreSQL)

*   **`LoyaltyTiers` Table (Optional, for tiered programs - keeping it simple for "basic")**
    *   If implemented, this table would define different tiers. For a basic non-tiered program, this table can be omitted, and all customers are effectively in a single implicit tier. We will proceed without this table for the "basic" design but note its potential.

*   **`Users` Table (Modifications - for `role = 'customer'`)**
    *   (Reference: `project_setup_plan.md`, `crm_module_design.md`)
    *   **New Fields:**
        *   `loyalty_points_balance` (INTEGER NOT NULL DEFAULT 0 CHECK (`loyalty_points_balance` >= 0))
        *   `marketing_opt_in` (BOOLEAN DEFAULT TRUE NOT NULL): Customer's preference for receiving marketing emails. This will also control sync to Mailchimp.
        *   `last_loyalty_point_activity_at` (TIMESTAMP WITH TIME ZONE, Nullable): To track last earn/redeem for points expiration logic.

    *SQL conceptual additions to `Users` table:*
    ```sql
    ALTER TABLE Users
    ADD COLUMN loyalty_points_balance INTEGER NOT NULL DEFAULT 0 CHECK (loyalty_points_balance >= 0),
    ADD COLUMN marketing_opt_in BOOLEAN DEFAULT TRUE NOT NULL,
    ADD COLUMN last_loyalty_point_activity_at TIMESTAMP WITH TIME ZONE;
    ```

*   **`LoyaltyPointsLedger` Table**
    *   Tracks all changes to a customer's loyalty points balance.
    *   **Schema:**
        *   `id` (SERIAL PRIMARY KEY)
        *   `customer_id` (INTEGER NOT NULL REFERENCES `Users`(`id`) ON DELETE CASCADE)
        *   `transaction_id` (INTEGER, REFERENCES `Transactions`(`id`) ON DELETE SET NULL, Nullable)
        *   `appointment_id` (INTEGER, REFERENCES `Appointments`(`id`) ON DELETE SET NULL, Nullable)
        *   `promotion_redemption_id` (INTEGER, REFERENCES `Promotions`(`id`) ON DELETE SET NULL, Nullable): If points were redeemed for a specific promotion.
        *   `points_earned` (INTEGER, Nullable CHECK (`points_earned` IS NULL OR `points_earned` > 0))
        *   `points_redeemed` (INTEGER, Nullable CHECK (`points_redeemed` IS NULL OR `points_redeemed` > 0))
        *   `reason_code` (TEXT NOT NULL, CHECK (`reason_code` IN ('purchase_earn', 'service_earn', 'reward_redemption', 'birthday_bonus', 'welcome_bonus', 'manual_adjustment_admin', 'points_expired', 'other')))
        *   `notes` (TEXT, Nullable): Additional details, e.g., "Manual adjustment by Admin X for issue Y".
        *   `created_at` (TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL)
        *   `expires_at` (TIMESTAMP WITH TIME ZONE, Nullable): If these specific points expire.

    *SQL for `LoyaltyPointsLedger` Table:*
    ```sql
    CREATE TABLE LoyaltyPointsLedger (
        id SERIAL PRIMARY KEY,
        customer_id INTEGER NOT NULL REFERENCES Users(id) ON DELETE CASCADE,
        transaction_id INTEGER REFERENCES Transactions(id) ON DELETE SET NULL,
        appointment_id INTEGER REFERENCES Appointments(id) ON DELETE SET NULL,
        promotion_redemption_id INTEGER REFERENCES Promotions(id) ON DELETE SET NULL,
        points_earned INTEGER CHECK (points_earned IS NULL OR points_earned > 0),
        points_redeemed INTEGER CHECK (points_redeemed IS NULL OR points_redeemed > 0),
        reason_code TEXT NOT NULL CHECK (reason_code IN ('purchase_earn', 'service_earn', 'reward_redemption', 'birthday_bonus', 'welcome_bonus', 'manual_adjustment_admin', 'points_expired', 'other')),
        notes TEXT,
        created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
        expires_at TIMESTAMP WITH TIME ZONE,
        CONSTRAINT fk_loyalty_customer FOREIGN KEY (customer_id) REFERENCES Users(id) ON DELETE CASCADE,
        CONSTRAINT fk_loyalty_transaction FOREIGN KEY (transaction_id) REFERENCES Transactions(id) ON DELETE SET NULL,
        CONSTRAINT fk_loyalty_appointment FOREIGN KEY (appointment_id) REFERENCES Appointments(id) ON DELETE SET NULL,
        CONSTRAINT fk_loyalty_promotion FOREIGN KEY (promotion_redemption_id) REFERENCES Promotions(id) ON DELETE SET NULL,
        CONSTRAINT check_points_earned_or_redeemed CHECK ((points_earned IS NOT NULL AND points_redeemed IS NULL) OR (points_earned IS NULL AND points_redeemed IS NOT NULL))
    );
    CREATE INDEX idx_loyaltyledger_customer_id ON LoyaltyPointsLedger(customer_id);
    CREATE INDEX idx_loyaltyledger_created_at ON LoyaltyPointsLedger(created_at);
    CREATE INDEX idx_loyaltyledger_expires_at ON LoyaltyPointsLedger(expires_at) WHERE expires_at IS NOT NULL;
    ```

*   **`Promotions` Table (Modifications - from `pos_enhancements_design.md`)**
    *   **New Fields:**
        *   `is_loyalty_reward` (BOOLEAN DEFAULT FALSE NOT NULL)
        *   `points_cost` (INTEGER, Nullable CHECK (`points_cost` IS NULL OR `points_cost` > 0)): Number of loyalty points required to redeem this promotion.

    *SQL conceptual additions to `Promotions` table:*
    ```sql
    ALTER TABLE Promotions
    ADD COLUMN is_loyalty_reward BOOLEAN DEFAULT FALSE NOT NULL,
    ADD COLUMN points_cost INTEGER CHECK (points_cost IS NULL OR points_cost > 0);
    ```

### 2. Logic (Backend - `LoyaltyService`)

*   **Earning Points:**
    *   **Configuration:** System settings (e.g., in a global `SalonSettings` table or config file):
        *   `points_per_dollar_spent_services` (e.g., 1 point per $1).
        *   `points_per_dollar_spent_products` (e.g., 2 points per $1).
        *   (Optional) Fixed points per specific service/product ID.
    *   **Trigger:** After a `Transaction` `payment_status` is 'succeeded' or 'paid_multiple'.
    *   **Process:**
        1.  For each `TransactionItem` in the transaction:
            *   If it's a service, calculate points based on `price_after_discount` and `points_per_dollar_spent_services`.
            *   If it's a product, calculate points based on `price_after_discount` and `points_per_dollar_spent_products`.
        2.  Sum total points earned for the transaction.
        3.  If total points > 0:
            *   Update `Users.loyalty_points_balance` (atomic increment).
            *   Update `Users.last_loyalty_point_activity_at = NOW()`.
            *   Create a `LoyaltyPointsLedger` entry: `customer_id`, `transaction_id`, `points_earned`, `reason_code = 'purchase_earn'`. (Can be one entry per transaction or per item).
    *   **Welcome Bonus:** When a new customer registers, optionally grant welcome bonus points and create a ledger entry (`reason_code = 'welcome_bonus'`).
    *   **Birthday Bonus:** A scheduled job (similar to birthday reminder job in CRM enhancements) can grant birthday bonus points and create a ledger entry (`reason_code = 'birthday_bonus'`).
*   **Redeeming Points:**
    *   **Trigger:** When a `Promotion` is applied to a transaction (`POST /api/pos/transactions` modified in `pos_enhancements_design.md`).
    *   **Process (within transaction processing logic):**
        1.  If the applied `Promotion` has `is_loyalty_reward = true` and `points_cost > 0`:
            *   Fetch customer's `loyalty_points_balance`.
            *   If `balance < points_cost`, reject the promotion application with an error "Insufficient loyalty points".
            *   If sufficient:
                *   Update `Users.loyalty_points_balance` (atomic decrement by `points_cost`).
                *   Update `Users.last_loyalty_point_activity_at = NOW()`.
                *   Create a `LoyaltyPointsLedger` entry: `customer_id`, `transaction_id`, `promotion_redemption_id = promotion.id`, `points_redeemed = points_cost`, `reason_code = 'reward_redemption'`.
*   **Points Expiration (Optional - Scheduled Job):**
    *   **Configuration:** System setting `loyalty_points_expiry_months` (e.g., 12 months).
    *   **Job Logic (e.g., daily `node-cron`):**
        1.  Find `LoyaltyPointsLedger` entries where `points_earned > 0`, `expires_at IS NULL` (or a flag `has_been_used_for_expiry_calc = false`), and `created_at < NOW() - interval 'X months'`.
        2.  Alternatively, a simpler model: if `Users.last_loyalty_point_activity_at < NOW() - interval 'X months'`, expire the entire balance. This is less fair if new points were earned recently.
        3.  A better model: For each customer, sum `points_earned` that are older than X months and haven't expired, and sum `points_redeemed`. If `earned_old_points > redeemed_points_total`, then some old points are eligible to expire.
        4.  For MVP, a simpler approach: Points expire if not used within X months of earning. The `LoyaltyPointsLedger.expires_at` can be set at creation (e.g., `created_at + interval 'X months'`). The job then finds entries where `expires_at <= NOW()` and `points_earned` are not fully offset by `points_redeemed` against them (complex to track perfectly without individual point IDs).
        5.  **Simplified MVP Expiry:** If `Users.last_loyalty_point_activity_at` is older than `loyalty_points_expiry_months`, set `loyalty_points_balance = 0` and create a ledger entry for `points_redeemed` (entire balance) with `reason_code = 'points_expired'`.

### 3. API Endpoints (Backend)

*   **`GET /api/customers/me/loyalty` (For logged-in customer)**
    *   Permissions: Authenticated Customer.
    *   Response (200 OK):
        ```json
        {
          "loyaltyPointsBalance": 550,
          "lastActivityAt": "YYYY-MM-DDTHH:mm:ss.sssZ",
          // "loyaltyTier": { "name": "Silver", "benefits_description": "..." } // If tiers implemented
          "history": [ // Paginated list from LoyaltyPointsLedger
            { "id": 1, "points_earned": 100, "reason_code": "purchase_earn", "transaction_id": 123, "created_at": "..." },
            { "id": 2, "points_redeemed": 50, "reason_code": "reward_redemption", "promotion_redemption_id": 10, "created_at": "..." }
          ],
          "availableRewards": [ // Filtered from Promotions where is_loyalty_reward = true and points_cost <= balance
            { "promotionId": 10, "name": "10% Off Next Service", "pointsCost": 50, "description": "..." }
          ]
        }
        ```
*   **`GET /api/customers/{customerId}/loyalty` (Admin/Staff view)**
    *   Permissions: Admin, Staff.
    *   Similar response to `/api/customers/me/loyalty`.
*   **`POST /api/loyalty/admin/adjust-points`**
    *   Permissions: Admin.
    *   Request Body: `{ "customerId": 123, "pointsChange": 50, "reason_code": "manual_adjustment_admin", "notes": "Compensation for service issue." }` (`pointsChange` can be negative).
    *   Logic:
        1.  Update `Users.loyalty_points_balance` by `pointsChange`.
        2.  Create `LoyaltyPointsLedger` entry (`points_earned` or `points_redeemed` based on sign of `pointsChange`).
    *   Response (200 OK): Updated loyalty balance for the customer.
*   **CRUD for `LoyaltyTiers` (Admin - if implemented):** Standard CRUD endpoints.

### 4. UI (Frontend)

*   **Customer Mobile App / Web Profile:**
    *   "My Loyalty" or "Rewards" section.
    *   Display `loyaltyPointsBalance`.
    *   List `history` (earned/redeemed, reason, date).
    *   List `availableRewards` (promotions they can afford with points).
*   **POS View (Staff):**
    *   When a customer is selected for a transaction:
        *   Display their `loyaltyPointsBalance` if available.
        *   When applying promotions, loyalty rewards (those with `points_cost`) should be clearly marked. If selected, the system checks points balance before applying.
*   **Admin Panel:**
    *   Section "Loyalty Program".
    *   Settings: Points earning rules (e.g., $1 = X points for services, $1 = Y points for products), welcome bonus points, birthday bonus points, points expiry rules.
    *   View customer loyalty data (search customer, see balance, ledger).
    *   "Adjust Points" form for manual changes.
    *   (If Tiers) Manage `LoyaltyTiers` (create, edit, delete tiers, define benefits).

## II. Basic Email Marketing Integration (e.g., Mailchimp)

### 1. System Setup (Backend)

*   **Configuration:**
    *   Admin provides Mailchimp API Key and default Audience (List) ID.
    *   Store these securely (e.g., environment variables or encrypted in a `SalonSettings` table).
*   **Library:** Use Mailchimp's official Node.js library (e.g., `@mailchimp/mailchimp_marketing`).

### 2. Logic (Backend - `MailchimpService`)

*   **Customer Sync (Add/Update Subscriber):**
    *   **Trigger:**
        *   After new customer creation (`POST /api/auth/register` or `POST /api/customers`).
        *   After customer email update (`PUT /api/users/me` or `PUT /api/customers/{customerId}`).
        *   After customer `marketing_opt_in` status changes.
    *   **Process:**
        1.  Check `Users.marketing_opt_in`.
        2.  If `true`:
            *   Prepare subscriber data: `email_address`, `status: 'subscribed'`, merge fields (e.g., `FNAME`, `LNAME`), tags (e.g., "SalonCustomer", "LoyaltyProgramMember").
            *   Call Mailchimp API to add/update subscriber in the specified Audience ID. Use `md5` hash of lowercase email for `subscriber_hash` if updating.
        3.  If `false`:
            *   Set `status: 'unsubscribed'` in Mailchimp for that subscriber.
*   **Batch Sync (Optional - for initial setup or recovery):**
    *   Admin-triggered job to iterate through all customers and sync them to Mailchimp based on their `marketing_opt_in` status.

### 3. API Endpoints (Backend)

*   **`POST /api/marketing/mailchimp/admin/sync-all-customers` (Admin-triggered batch sync)**
    *   Permissions: Admin.
    *   Logic: Iterates through users and calls `MailchimpService.syncCustomer` for each.
    *   Response: `{ "success": true, "message": "Batch sync initiated." }` (as this is a background job).
*   No direct customer-facing API for Mailchimp is usually needed. Actions are triggered by other events.

### 4. UI (Frontend)

*   **Customer Profile/Registration (Mobile App & Web):**
    *   Checkbox: "I want to receive marketing emails and promotions." (Bound to `Users.marketing_opt_in`).
    *   Default can be true or false based on local regulations (e.g., GDPR requires opt-in, so default false).
*   **Admin Panel:**
    *   Section "Marketing Settings" or "Integrations".
    *   Input fields for Mailchimp API Key and Audience ID.
    *   "Test Connection" button.
    *   "Sync All Customers to Mailchimp" button (triggers batch sync API).
    *   Display sync status/logs (basic).

## III. Deliverables Summary

*   **Database Schema Changes/Additions:**
    *   Modifications to `Users` table (`loyalty_points_balance`, `marketing_opt_in`, `last_loyalty_point_activity_at`).
    *   New `LoyaltyPointsLedger` table.
    *   Modifications to `Promotions` table (`is_loyalty_reward`, `points_cost`).
    *   (Optional) `LoyaltyTiers` table.
*   **Logic Outlines:**
    *   Loyalty points earning rules and process.
    *   Loyalty points redemption process (linked with POS promotions).
    *   Optional points expiration logic.
    *   Mailchimp customer data synchronization (add/update subscriber, opt-out handling).
*   **API Designs:**
    *   Loyalty: `GET /api/customers/me/loyalty` (customer view), `GET /api/customers/{customerId}/loyalty` (admin view), `POST /api/loyalty/admin/adjust-points`.
    *   Mailchimp: (Mostly internal triggers) `POST /api/marketing/mailchimp/admin/sync-all-customers` for batch sync.
*   **UI/UX Outlines:**
    *   Customer views for loyalty points, history, and available rewards.
    *   POS integration for displaying points and applying loyalty rewards.
    *   Admin views for managing loyalty program settings, customer loyalty data, and Mailchimp configuration.
    *   Customer opt-in checkbox for marketing communications.

This design provides a comprehensive plan for the Basic Marketing & Loyalty Programs.
