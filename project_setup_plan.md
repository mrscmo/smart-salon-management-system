# Salon Management System - Project Setup and Authentication Plan

This document outlines the key decisions, tasks, and deliverables for setting up the project structure and user authentication for the Salon Management System.

## 1. Technology Stack Selection

After consideration of development speed, performance, scalability, available libraries, and ease of learning, the following technology stack has been chosen:

*   **Backend:** **Node.js with Express.js**
    *   **Justification:** Node.js with Express.js offers rapid development, a vast ecosystem of packages via npm, and performance suitable for a web application of this nature. Its non-blocking I/O is well-suited for handling multiple client requests simultaneously. Using JavaScript on the backend also allows for language consistency if the frontend is JavaScript-based.
*   **Frontend:** **React**
    *   **Justification:** React's component-based architecture promotes reusability and maintainability. It has a large community, excellent state management libraries (like Redux or Zustand), and a strong ecosystem. Its declarative nature makes UI development more predictable.
*   **Database:** **PostgreSQL**
    *   **Justification:** PostgreSQL is a powerful, open-source object-relational database system known for its reliability, data integrity, and rich feature set. It supports complex queries, JSONB for flexible data types, and scales well.
*   **Mobile (Future Consideration):**
    *   **Potential Technology:** **React Native**
    *   **Influence of Backend Choice:** The choice of Node.js for the backend is conducive to React Native development, as both leverage JavaScript. APIs developed for the web application can be readily consumed by a React Native mobile app.

## 2. Project Initialization & Structure

*   **Version Control:**
    *   A Git repository will be initialized.
    *   **Branching Strategy:** Gitflow will be adopted (main, develop, feature/*, release/*, hotfix/*).
*   **Backend Project Structure (Node.js/Express.js):**
    ```
    salon-backend/
    ├── src/
    │   ├── config/         # Environment variables, database config
    │   ├── controllers/    # Request handlers
    │   ├── middleware/     # Custom middleware (e.g., auth, error handling)
    │   ├── models/         # Database models/schemas (e.g., Mongoose, Sequelize)
    │   ├── routes/         # API route definitions
    │   ├── services/       # Business logic
    │   ├── utils/          # Utility functions
    │   └── app.js          # Express app initialization
    ├── tests/              # Unit and integration tests
    ├── .env.example        # Example environment variables
    ├── .gitignore
    ├── package.json
    └── README.md
    ```
*   **Frontend Project Structure (React):**
    ```
    salon-frontend/
    ├── public/             # Static assets, index.html
    ├── src/
    │   ├── api/            # API service calls
    │   ├── assets/         # Images, fonts, etc.
    │   ├── components/     # Reusable UI components (dumb components)
    │   │   ├── common/     # Buttons, Inputs, Modals etc.
    │   │   └── layout/     # Navbar, Footer, Sidebar etc.
    │   ├── contexts/       # React Context API for state management (if not using Redux/Zustand)
    │   ├── hooks/          # Custom React hooks
    │   ├── pages/          # Top-level route components (smart components)
    │   ├── services/       # Business logic specific to frontend (e.g. local storage)
    │   ├── store/          # State management (e.g., Redux, Zustand)
    │   ├── styles/         # Global styles, themes
    │   ├── utils/          # Utility functions
    │   └── App.js          # Main application component
    │   └── index.js        # Entry point
    ├── .env.example
    ├── .gitignore
    ├── package.json
    └── README.md
    ```
*   **Environment Setup:**
    *   Development, testing, and production environments will be managed using `.env` files.
    *   `dotenv` library (or similar) will be used to load environment variables.
    *   Future consideration: Docker for containerization to ensure consistency across environments.

## 3. User Authentication & Authorization

*   **User Model/Schema (`Users` table):**
    *   `id` (Primary Key, e.g., UUID or auto-incrementing integer)
    *   `email` (String, Unique, Indexed)
    *   `password_hash` (String)
    *   `first_name` (String)
    *   `last_name` (String)
    *   `phone_number` (String, Optional)
    *   `role` (Enum/String: 'admin', 'staff', 'customer', default: 'customer')
    *   `is_verified` (Boolean, default: false) - For email verification
    *   `created_at` (Timestamp)
    *   `updated_at` (Timestamp)
*   **Registration:**
    *   **Backend API Endpoint:** `POST /api/auth/register`
        *   Request Body: `email`, `password`, `firstName`, `lastName`, `phoneNumber` (optional)
        *   Response: User object (excluding password) or success message.
        *   Action: Hashes password, creates new user record. Email verification (optional at this stage, but planned): sends a verification email.
    *   **Frontend:** Registration form with fields for email, password, password confirmation, first name, last name.
*   **Login:**
    *   **Backend API Endpoint:** `POST /api/auth/login`
        *   Request Body: `email`, `password`
        *   Response: JWT (JSON Web Token) and user information (excluding password).
        *   Action: Verifies credentials, issues JWT.
    *   **Frontend:** Login form with email and password fields. Stores JWT in `localStorage` or `sessionStorage` (or HttpOnly cookie for better security).
*   **Password Management:**
    *   **Hashing:** `bcrypt` or `Argon2` will be used for hashing passwords. Salting will be incorporated.
    *   **Password Reset:** (Planned, can be deferred slightly)
        *   Endpoint: `POST /api/auth/request-password-reset` (takes email)
        *   Endpoint: `POST /api/auth/reset-password` (takes token and new password)
*   **Role Management:**
    *   Roles: `admin`, `staff`, `customer`.
    *   Mechanism:
        *   The `User` model will have a `role` field.
        *   Backend middleware will protect routes. For example:
            *   `ensureAuthenticated`: Checks for a valid JWT.
            *   `ensureRole(['admin'])`: Checks if the user has the 'admin' role.
            *   `ensureRole(['admin', 'staff'])`: Checks if the user is 'admin' or 'staff'.
        *   Frontend will conditionally render UI elements based on the user's role.

## 4. Database Schema (Core Entities - Initial Draft)

*   **`Users`:** (Covered in Authentication section)
*   **`Salons`:** (Simplified for initial setup, assuming single salon initially. Can be expanded for multi-tenancy later)
    *   `id` (PK)
    *   `name` (String)
    *   `address` (String)
    *   `phone_number` (String)
    *   `operating_hours` (JSONB or Text)
    *   `created_at` (Timestamp)
    *   `updated_at` (Timestamp)
*   **`Services`:**
    *   `id` (PK)
    *   `salon_id` (FK referencing `Salons.id`, if multi-salon) - For now, can be omitted if single salon.
    *   `name` (String)
    *   `description` (Text)
    *   `duration_minutes` (Integer)
    *   `price` (Decimal)
    *   `is_active` (Boolean, default: true)
    *   `created_at` (Timestamp)
    *   `updated_at` (Timestamp)
*   **`Appointments`:**
    *   `id` (PK)
    *   `customer_id` (FK referencing `Users.id` where role is 'customer')
    *   `staff_id` (FK referencing `Users.id` where role is 'staff')
    *   `service_id` (FK referencing `Services.id`)
    *   `start_time` (Timestamp)
    *   `end_time` (Timestamp)
    *   `status` (Enum/String: 'scheduled', 'confirmed', 'completed', 'cancelled_by_customer', 'cancelled_by_salon', 'no_show', default: 'scheduled')
    *   `notes` (Text, optional)
    *   `created_at` (Timestamp)
    *   `updated_at` (Timestamp)
*   **`Customers`:** (This entity might be merged into `Users` or act as a profile extension if more customer-specific, non-auth data is needed. For now, keeping it simple and assuming most data is in `Users`.)
    *   `user_id` (PK, FK referencing `Users.id`)
    *   `preferences` (Text, placeholder for customer preferences)
    *   `last_visit_date` (Date, optional)
*   **`Staff`:** (Similar to `Customers`, this can be an extension of the `Users` table for staff-specific attributes.)
    *   `user_id` (PK, FK referencing `Users.id`)
    *   `skills` (JSONB or Text Array, placeholder, e.g., ['cutting', 'coloring'])
    *   `availability_schedule` (JSONB, placeholder, e.g., defining working hours/days)
    *   `is_active` (Boolean, default: true)

**Relationships (Initial):**
*   A `User` can be a `Customer` or `Staff` (or `Admin`). The `role` field in `Users` handles this.
*   An `Appointment` connects a `Customer` (User), `Staff` (User), and a `Service`.
*   If multi-salon becomes a feature, `Services` would belong to a `Salon`. `Staff` might also be associated with specific `Salons`.

## Deliverables (Conceptual for this Subtask)

*   **This document (`project_setup_plan.md`)**: Outlines the chosen technology stack with justifications, project structure, authentication/authorization strategy, and initial database schema.
*   **API Endpoint Definitions (as part of this document):** Registration and login endpoints are defined in section 3.
*   **Database Schema Description (as part of this document):** The initial schema for core entities is described in section 4. A separate SQL script or diagram can be generated from this.
*   **Role-Based Access Control (RBAC) Description (as part of this document):** The RBAC mechanism is described in section 3.

This plan provides a foundational outline. Specific implementation details will be elaborated upon during the development sprints for each component.
