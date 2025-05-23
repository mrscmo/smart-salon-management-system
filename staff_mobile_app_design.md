# Staff Mobile App - UI/UX Concepts and API Requirements

This document outlines the UI/UX concepts and backend API requirements for the core features of a Staff Mobile App, built with React Native. It focuses on schedule viewing and customer information access, referencing existing API designs.

## I. UI/UX Design Concepts (React Native)

The app will feature a clean, intuitive interface, prioritizing ease of use for staff on the go. Navigation will primarily be handled by a bottom tab navigator.

### 1. Login Screen

*   **Elements:**
    *   Salon Logo.
    *   Email Input Field (placeholder: "Email").
    *   Password Input Field (placeholder: "Password", secure text entry).
    *   "Login" Button (primary action).
    *   "Forgot Password?" Link (navigates to a password reset flow - out of scope for initial core features, but link should be present).
*   **Behavior:**
    *   On successful login, store JWT and navigate to Main Dashboard.
    *   Display error messages for invalid credentials or server issues.

### 2. Main Dashboard/Home Screen (Post-Login)

*   **Purpose:** Provide a quick glance at the most relevant information for the day.
*   **Layout:**
    *   Header: "Welcome, [Staff Name]!"
    *   **Summary Section:**
        *   Card: "Today's Appointments"
            *   Shows: Total count of appointments for today.
            *   Shows: Details of the "Next Upcoming Appointment" (Time, Customer Name, Service). Tap to go to Appointment Details.
            *   Link/Button: "View Full Schedule" (navigates to Schedule View for today).
        *   (Future: Quick access to clock-in/out if that feature is added).
    *   **Navigation (Bottom Tab Navigator):**
        *   **Schedule Tab:** Icon (e.g., calendar). Default selected tab.
        *   **Customers Tab:** Icon (e.g., group of people).
        *   **Profile Tab:** Icon (e.g., person/settings).

### 3. Schedule View

*   **Accessed via:** "Schedule" tab in the bottom navigator.
*   **Default View: Daily List**
    *   Header: Date display (e.g., "Today, October 26").
    *   Navigation: Arrows or swipe gesture for "< Previous Day" and "Next Day >". A calendar icon to jump to a specific date.
    *   **Appointment List:**
        *   Chronological list of appointments for the logged-in staff member for the selected day.
        *   Each list item (Card):
            *   Start Time - End Time (e.g., "09:00 AM - 10:30 AM").
            *   Customer Name.
            *   Service Name.
            *   Status indicator (e.g., small colored dot for 'Confirmed', 'Completed', 'No-Show').
        *   Tap on an item to navigate to the `Appointment Details Screen`.
        *   Message if no appointments: "No appointments scheduled for this day."
*   **Optional View: Weekly Calendar (Future Enhancement)**
    *   A toggle to switch to a compact weekly calendar view.
    *   Days of the week show indicators if appointments are present. Tap a day to load its list below or switch to daily view.

### 4. Appointment Details Screen

*   **Accessed by:** Tapping an appointment from the Schedule View.
*   **Layout:**
    *   Header: "Appointment Details". Back button to Schedule View.
    *   **Key Information Section:**
        *   Date & Time: "October 26, 09:00 AM - 10:30 AM"
        *   Service: "Deluxe Haircut & Color" (Duration: 90 mins)
        *   Price: "$150.00" (Display only, no POS functions here)
    *   **Customer Information Section:**
        *   Customer Name: "Jane Doe" (Tap to navigate to `Customer Details Screen`).
        *   Contact: "Phone: 123-456-7890" (Tap to call - OS dependent).
    *   **Notes Section:**
        *   **Customer Notes (from CRM):**
            *   "Allergies: PPD (High Severity - Skin Rash). Avoid dark hair dyes."
            *   "Preferences: Prefers silent appointments."
        *   **Service Notes/Instructions (from Service definition):**
            *   "Ensure patch test was completed 24h prior for new color clients."
    *   **Status Update Section (Conditional based on permissions):**
        *   Current Status: "Confirmed"
        *   Buttons: "Mark as Completed", "Mark as No-Show".
            *   Requires confirmation dialog.
            *   Calls existing `PUT /api/appointments/{appointmentId}`.

### 5. Customer Information Access

*   **Accessed via:** "Customers" tab in the bottom navigator.
*   **Customer Search Screen:**
    *   Header: "Find Customer".
    *   Search Bar: Input field for Name or Phone Number.
    *   Real-time search results appear below as user types.
*   **Customer List View (Search Results):**
    *   Each item: Customer Name, Phone Number.
    *   Tap an item to navigate to `Customer Details Screen`.
    *   Message if no results: "No customers found."
*   **Customer Details Screen:**
    *   Header: Customer Name. Back button to Search/List.
    *   **Contact Information Section:**
        *   Full Name: "Jane Doe"
        *   Email: "jane.doe@example.com"
        *   Phone: "123-456-7890" (Tap to call)
    *   **Key Notes Section (from CRM):**
        *   "Allergies: PPD (High Severity - Skin Rash). Avoid dark hair dyes."
        *   "Preferences: Prefers silent appointments."
    *   **Appointments Section (with this Staff Member):**
        *   **Upcoming:** List of future appointments (Date, Time, Service). Tap to go to `Appointment Details Screen`.
        *   **Past History (Summary):** List of past 5-10 appointments (Date, Service, Status e.g. Completed/No-Show).

### 6. Profile/Settings Screen

*   **Accessed via:** "Profile" tab in the bottom navigator.
*   **Layout:**
    *   Header: "My Profile".
    *   **Staff Information (Read-only for now):**
        *   Name: "[Staff Full Name]"
        *   Email: "[Staff Email]"
        *   Skills: (List of skills, e.g., "Hair Cutting, Coloring, Styling") - from `Users.skill_sets`.
    *   **Actions:**
        *   "Logout" Button: Clears stored JWT, navigates to Login Screen.
    *   (Future: Manage Availability, Notification Preferences, Link to Google Calendar Sync).

## II. Backend API Requirements (Leverage & Extend Existing)

### 1. Authentication

*   **`POST /api/auth/login` (Existing):**
    *   No changes strictly needed if it returns JWT and user info (including `staffId` and role).
    *   **Mobile Consideration:** Ensure JWTs are not overly short-lived. Implement a **token refresh mechanism**:
        *   If mobile app receives a 401 error, it should attempt to use a refresh token (if issued during login) to get a new JWT.
        *   New Endpoint: `POST /api/auth/refresh-token` (takes refresh token, returns new JWT).
        *   If refresh fails, then log out user.
*   **`GET /api/users/me` (Conceptual - if not already present):**
    *   To get the logged-in staff member's own profile details (name, email, skills) for the Profile screen.
    *   Permissions: Authenticated User (self).
    *   Response: `{ "id": X, "firstName": "...", "lastName": "...", "email": "...", "skillSets": ["..."] }`

### 2. Staff Schedule & Appointments

*   **`GET /api/staff/{staffId}/appointments/mobile` (New or Enhanced)**
    *   Path: `/api/staff/{staffId}/appointments/mobile?date=YYYY-MM-DD`
    *   Alternative: Enhance existing `GET /api/appointments` with `staffId={staffId}&date={date}&view=mobile_daily_summary`.
    *   **Purpose:** Optimized for the mobile daily schedule view for a specific staff member.
    *   **Response:** Array of appointment objects, each tailored for mobile:
        ```json
        [
          {
            "appointmentId": 101,
            "startTime": "YYYY-MM-DDTHH:mm:ssZ",
            "endTime": "YYYY-MM-DDTHH:mm:ssZ",
            "customerName": "Jane Doe",
            "serviceName": "Women's Haircut",
            "status": "Confirmed" // e.g., "Confirmed", "Completed", "No-Show"
          }
          // ... more appointments
        ]
        ```
*   **`GET /api/appointments/{appointmentId}` (Existing - Ensure Completeness)**
    *   **Purpose:** For the "Appointment Details Screen".
    *   **Ensure Response includes:**
        *   Full appointment details (date, time, service name, duration, price).
        *   Customer ID, Customer Name, Customer Phone.
        *   Key Customer Notes (relevant `allergies_sensitivities` from `Users` table, general notes from `Users.notes`).
        *   Service-specific notes/instructions (from `Services.requirements` or `Services.description`).
        *   Current `status`.
        ```json
        // Example of expected structure
        {
          "appointmentId": 101,
          "startTime": "YYYY-MM-DDTHH:mm:ssZ",
          "endTime": "YYYY-MM-DDTHH:mm:ssZ",
          "serviceName": "Deluxe Haircut & Color",
          "serviceDurationMinutes": 90,
          "servicePrice": "150.00",
          "serviceRequirements": "Ensure patch test was completed 24h prior...",
          "status": "Confirmed",
          "customer": {
            "customerId": 123,
            "customerName": "Jane Doe",
            "customerPhoneNumber": "123-456-7890",
            "notes": { // Consolidated customer notes for easy display
              "allergies_sensitivities": [
                {"allergen": "PPD", "reaction_type": "Skin Rash", "severity": "High"}
              ],
              "preferences": "Prefers silent appointments."
            }
          }
        }
        ```

### 3. Customer Information

*   **`GET /api/customers/search/mobile` (New or Enhanced)**
    *   Path: `/api/customers/search/mobile?query={search_term}`
    *   Alternative: Enhance existing `GET /api/customers?search={query}&view=mobile_list`.
    *   **Purpose:** Optimized for mobile customer search (name, phone).
    *   **Response:** Lightweight array of customer objects:
        ```json
        [
          {
            "customerId": 123,
            "name": "Jane Doe",
            "phoneNumber": "123-456-7890"
          }
          // ... more customers
        ]
        ```
*   **`GET /api/customers/{customerId}/mobile-staff-view` (New or Enhanced)**
    *   Path: `/api/customers/{customerId}/mobile-staff-view?staffId={loggedInStaffId}`
    *   Alternative: Enhance existing `GET /api/customers/{customerId}` with query params like `?view=mobile_staff&forStaffId={staffId}`.
    *   **Purpose:** Get customer details tailored for staff mobile app.
    *   **Response:**
        ```json
        {
          "customerId": 123,
          "name": "Jane Doe",
          "email": "jane.doe@example.com",
          "phoneNumber": "123-456-7890",
          "notes": { // Consolidated customer notes
            "allergies_sensitivities": [ /* ... */ ],
            "preferences": "..."
          },
          "upcomingAppointmentsWithStaff": [ // Only appointments with the requesting staff member
            { "appointmentId": 205, "date": "YYYY-MM-DD", "time": "HH:MM", "serviceName": "Follow-up Consultation" }
          ],
          "pastAppointmentSummaryWithStaff": [ // Summary, only with this staff
            { "date": "YYYY-MM-DD", "serviceName": "Initial Cut", "status": "Completed" }
          ]
        }
        ```

### 4. Appointment Status Updates

*   **`PUT /api/appointments/{appointmentId}` (Existing):**
    *   Use as is. Mobile app would send `{ "status": "Completed" }` or `{ "status": "No-Show" }`.
    *   Backend must enforce permissions (ensure the logged-in staff is associated with the appointment or has general permission to update status).

## III. Push Notifications (Conceptual)

While full implementation is for future iterations, API considerations for enabling push notifications include:

*   **`POST /api/users/me/device-token` (New):**
    *   When app starts or user logs in, client sends device push token (FCM/APNS) to backend.
    *   Request: `{ "token": "device_push_token_here", "device_type": "android" | "ios" }`
    *   Backend stores this token associated with the user/staff member.
*   **Backend Logic:** When an event occurs (e.g., new appointment for staff, appointment change), the backend logic responsible for that event would also:
    1.  Identify affected staff members.
    2.  Retrieve their stored device push tokens.
    3.  Construct a notification payload.
    4.  Send the payload via a push notification service (e.g., Firebase Admin SDK, APNS library) to the relevant devices.

## IV. Deliverables Summary

*   **UI/UX Concepts:** High-level descriptions and key elements for core screens (Login, Dashboard, Schedule, Appointment Details, Customer Search/List/Details, Profile).
*   **Backend API Requirements:**
    *   Authentication: Confirmation of existing login, need for token refresh (`POST /api/auth/refresh-token`), and staff profile (`GET /api/users/me`).
    *   Staff Schedule: Optimized endpoint for daily view (`GET /api/staff/{staffId}/appointments/mobile`).
    *   Appointment Details: Ensured `GET /api/appointments/{appointmentId}` provides comprehensive data including customer/service notes.
    *   Customer Info: Optimized search (`GET /api/customers/search/mobile`) and tailored detail view (`GET /api/customers/{customerId}/mobile-staff-view`).
    *   Status Updates: Use of existing `PUT /api/appointments/{appointmentId}`.
*   **Push Notifications:** Conceptual notes on device token registration API and backend trigger logic.

This design provides a foundational plan for the Staff Mobile App's core features.
