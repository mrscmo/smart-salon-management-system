# Customer Mobile App - UI/UX Concepts and API Requirements

This document outlines the UI/UX concepts and backend API requirements for the core features of a Customer Mobile App, built with React Native. It focuses on service browsing, appointment booking, and managing appointments. It references existing API designs.

## I. UI/UX Design Concepts (React Native)

The app will prioritize a simple, visually appealing, and intuitive user experience, making it easy for customers to manage their salon interactions. Navigation will primarily use a bottom tab navigator.

### 1. Login/Registration Screen

*   **Elements:**
    *   Salon Logo.
    *   Segmented control or Tabs: "Login" | "Sign Up".
    *   **Login Tab (Default):**
        *   Email Input Field.
        *   Password Input Field (secure text entry).
        *   "Login" Button.
        *   "Forgot Password?" Link.
    *   **Sign Up Tab:**
        *   First Name Input Field.
        *   Last Name Input Field.
        *   Email Input Field.
        *   Phone Number Input Field (Optional but recommended for SMS).
        *   Password Input Field.
        *   Confirm Password Input Field.
        *   "Sign Up" Button.
        *   Link to Terms & Conditions / Privacy Policy.
    *   **Social Login (Conceptual - Below form fields):**
        *   Buttons: "Continue with Google", "Continue with Facebook".
*   **Behavior:**
    *   On successful login/registration, store JWT and navigate to Main Dashboard.
    *   Display error messages for invalid input, credentials, or server issues.

### 2. Main Dashboard/Home Screen (Post-Login)

*   **Purpose:** Welcome users, provide quick access to common actions, and optionally display promotions.
*   **Layout:**
    *   Header: "Welcome, [Customer Name]!" or "Good Morning, [Customer Name]!".
    *   **Quick Actions Section (Prominent Cards/Buttons):**
        *   "Book New Appointment" (navigates to Service Listing).
        *   "My Upcoming Appointments" (navigates to My Appointments > Upcoming).
    *   **Promotions/Featured Section (Optional - Carousel or Cards):**
        *   Display featured services or current salon promotions. Tap to view details.
    *   **Navigation (Bottom Tab Navigator):**
        *   **Home Tab:** Icon (e.g., house). Default selected tab.
        *   **Book Tab:** Icon (e.g., scissors, calendar plus).
        *   **Appointments Tab:** Icon (e.g., calendar list).
        *   **Profile Tab:** Icon (e.g., person).

### 3. Service Menu & Booking Flow

*   **Accessed via:** "Book" tab.
*   **Service Listing Screen:**
    *   Header: "Our Services".
    *   Search Bar: Filter services by name.
    *   Category Filter: Tabs or dropdown to filter by service category (e.g., "Hair", "Nails", "Skin").
    *   **Service List:**
        *   Scrollable list of services.
        *   Each item (Card): Service Name, short description snippet, Price, Duration.
        *   Tap on a service to navigate to `Service Details Screen` (or directly to Date/Time selection if details are minimal).
*   **Service Details Screen (Optional - if more details needed before booking):**
    *   Header: Service Name. Back button.
    *   Full Description, Duration, Price, "Requirements" (if any).
    *   Button: "Book This Service" (navigates to Date & Time Selection).
*   **Date & Time Selection Screen:**
    *   Header: "Select Date & Time" (Service Name displayed for context).
    *   **Date Picker:** Calendar view to select a date.
    *   **(Optional) Staff Selector:**
        *   Dropdown or horizontal scroll list: "Any Available Staff" (default), or list of staff names who perform the selected service.
    *   **Available Time Slots:**
        *   Displayed as a list or grid of buttons for the selected date and staff preference.
        *   Fetches from `GET /api/availability`.
        *   Indicates if a slot is nearly full or popular (future).
        *   Message if no slots: "No available slots for this day/staff. Please try another selection."
*   **Booking Confirmation Screen:**
    *   Header: "Confirm Your Booking".
    *   **Summary Section:**
        *   Service: [Service Name]
        *   Date: [Selected Date]
        *   Time: [Selected Time Slot]
        *   Staff: [Selected Staff Name or "Any Available"]
        *   Price: [Service Price]
        *   (Optional) Add notes for the salon.
    *   **Button: "Confirm & Book"** (primary action).
*   **Booking Success Screen:**
    *   Header: "Booking Confirmed!"
    *   Message: "Your appointment for [Service Name] on [Date] at [Time] is confirmed."
    *   Options: "View My Appointments", "Book Another Service".

### 4. My Appointments Screen

*   **Accessed via:** "Appointments" tab.
*   **Layout:** Segmented control/Tabs: "Upcoming" | "Past".
*   **Upcoming Appointments Tab (Default):**
    *   List of appointments (chronological).
    *   Each item (Card):
        *   Date & Time.
        *   Service Name.
        *   Staff Name.
        *   Status (e.g., "Confirmed").
        *   Buttons/Options:
            *   "View Details" (expands or navigates to a detail view with full info, salon address, map link - future).
            *   "Cancel Appointment" (triggers confirmation, then API call).
            *   "Reschedule" (MVP: may link to phone number or a request form; Advanced: re-runs booking flow with current service pre-selected).
    *   Message if no upcoming appointments: "You have no upcoming appointments."
*   **Past Appointments Tab:**
    *   List of past appointments (reverse chronological).
    *   Each item (Card): Date, Service Name, Staff Name, Status (e.g., "Completed", "Cancelled", "No-Show").
    *   Button/Option: "Book Again" (pre-selects the same service and potentially staff, then navigates to Date & Time Selection).
    *   Message if no past appointments.

### 5. Profile Screen

*   **Accessed via:** "Profile" tab.
*   **Layout:**
    *   Header: "My Profile".
    *   **User Information Section:**
        *   Name: "[Customer Name]" (Tap to edit - navigates to Edit Profile Screen).
        *   Email: "[Customer Email]" (Tap to edit).
        *   Phone: "[Customer Phone]" (Tap to edit).
    *   **Preferences Section:**
        *   "Notification Preferences" (navigates to a screen to toggle Email/SMS for reminders, confirmations - if granular control is implemented).
    *   **Payment Methods Section (Conceptual for now):**
        *   "Manage Payment Methods" (would show saved payment methods - display only for MVP).
    *   **Actions Section:**
        *   "Logout" Button.
*   **Edit Profile Screen:**
    *   Form fields for First Name, Last Name, Email, Phone Number.
    *   "Save Changes" button.

## II. Backend API Requirements (Leverage & Extend Existing)

### 1. Authentication & Registration

*   **`POST /api/auth/login` (Existing):**
    *   Use as is. Ensure JWT includes `customerId` (user ID) and `role`.
*   **`POST /api/auth/register` (Existing or Enhanced):**
    *   Review to ensure it's suitable for public customer self-registration.
    *   Request: `{ firstName, lastName, email, password, phoneNumber (optional) }`.
    *   Backend should assign `role: 'customer'` by default.
*   **Token Refresh Mechanisms (Essential for Mobile):**
    *   If not already designed, need `POST /api/auth/refresh-token`.
*   **`GET /api/users/me` (Existing or Conceptual):**
    *   To fetch the logged-in customer's own profile data for the Profile screen.
    *   Response: `{ "id": X, "firstName": "...", "lastName": "...", "email": "...", "phoneNumber": "...", "birth_date": "...", "anniversary_date": "...", "allergies_sensitivities": [...], "sms_notifications_enabled": true/false, ... }`

### 2. Service Menu & Booking

*   **`GET /api/services` (Existing):**
    *   Ensure query parameters `?is_active=true` and `?category={category_name}` are robust.
    *   Mobile might benefit from pagination if list is very long.
    *   Payload for list items should be concise: `id, name, short_description, price, duration_minutes, category`.
*   **`GET /api/services/{serviceId}` (Existing):**
    *   Use as is for service details.
*   **`GET /api/availability` (Existing - from `appointment_module_design.md`):**
    *   Path: `/api/availability?serviceId=X&date=YYYY-MM-DD&staffId={optionalStaffId}`
    *   Ensure this API efficiently returns available slots, considering service duration, staff schedules (and `StaffServices` linkage), existing appointments, and overrides.
    *   The `staffId` parameter should be optional. If not provided, the API should return slots for any available staff qualified for the service.
*   **`POST /api/appointments/book` (Existing):**
    *   The backend should automatically use the `customerId` from the authenticated JWT.
    *   Request from mobile: `{ serviceId, staffId (optional), startTime, notes (optional) }`.

### 3. Customer Appointments Management

*   **`GET /api/customers/me/appointments` (New or Enhanced `GET /api/appointments`)**
    *   **Purpose:** A unified endpoint for the logged-in customer to fetch their appointments, differentiated by status.
    *   Path: `/api/customers/me/appointments?status_group=upcoming` or `/api/customers/me/appointments?status_group=past`
    *   Query Parameters:
        *   `status_group`: "upcoming" (scheduled, confirmed), "past" (completed, cancelled, no-show).
        *   `page`, `limit` for pagination.
    *   **Response (Optimized for Mobile List View):** Array of appointment objects.
        ```json
        [
          {
            "appointmentId": 101,
            "serviceName": "Women's Haircut",
            "date": "YYYY-MM-DD",
            "startTime": "HH:MM", // Formatted time string
            "endTime": "HH:MM",   // Formatted time string
            "staffName": "Jane Doe", // Or "Any Available" if booked that way
            "status": "Confirmed" // or "Completed", "Cancelled"
            // Potentially salon branch details if multi-branch
          }
        ]
        ```
*   **`PUT /api/appointments/{appointmentId}/cancel` (New or Modification of `DELETE /api/appointments/{appointmentId}`)**
    *   **Purpose:** Customer cancels their own appointment.
    *   Backend must:
        1.  Verify the logged-in customer owns the appointment.
        2.  Check cancellation policies (e.g., `cancellation_cutoff_hours` from `BookingPolicies` or `SalonSettings` table). If policy violated, return an error (e.g., 400 Bad Request "Cancellation period has passed").
        3.  Update `Appointments.status` to 'cancelled_by_customer'.
    *   Response: (200 OK) with updated appointment object or (204 No Content).
*   **Reschedule (MVP approach):**
    *   For MVP, direct reschedule might be too complex. Instead, the "Reschedule" button could:
        1.  Suggest cancelling and rebooking.
        2.  Provide salon contact information to reschedule manually.
    *   If a more integrated reschedule is desired later:
        *   `PUT /api/appointments/{appointmentId}/reschedule`: Request `{ newStartTime, newStaffId (optional) }`. Backend would need to re-validate availability and conflicts.

### 4. User Profile Management

*   **`PUT /api/users/me` (Existing or Enhanced `PUT /api/customers/{customerId}`):**
    *   Allows customer to update their own profile information.
    *   Request: `{ firstName, lastName, email, phoneNumber, birth_date, anniversary_date, allergies_sensitivities, sms_notifications_enabled }`.
    *   Backend must only allow updating fields relevant to the user's own profile and ensure proper validation (e.g., email uniqueness if changed).

## III. Push Notifications (Conceptual - from `staff_mobile_app_design.md`)

Leverage existing conceptual design for push notifications:
*   Client app registers device token with `POST /api/users/me/device-token`.
*   Backend triggers notifications for:
    *   Appointment Confirmation (after `POST /api/appointments/book`).
    *   Appointment Reminders (via scheduled job).
    *   Notification of successful cancellation/reschedule.
    *   (Future) Promotional messages if opted-in.

## IV. Deliverables Summary

*   **UI/UX Concepts:** High-level descriptions and key elements for core screens of the Customer Mobile App (Login/Registration, Dashboard, Service Menu & Booking Flow, My Appointments, Profile).
*   **Backend API Requirements:**
    *   Authentication & Registration: Confirmation of existing login/registration, need for token refresh, and customer self-profile (`GET /api/users/me`).
    *   Service Menu & Booking: Use of existing service and availability APIs, ensuring support for optional staff preference.
    *   Customer Appointments: New optimized endpoint `GET /api/customers/me/appointments?status_group=...` and an endpoint for customer-initiated cancellation (`PUT /api/appointments/{appointmentId}/cancel`) with policy checks.
    *   User Profile Management: Use of `PUT /api/users/me` for self-service profile updates.
*   **Push Notifications:** Referenced existing conceptual design.
*   **Considerations:** Notes on customer self-registration and a phased approach to appointment modification (cancel first, then rebook for MVP reschedule).

This design provides a foundational plan for the Customer Mobile App's core features.
