This document outlines the technical specifications and user flow for an Invitation-Only Social Login System, tailored to the project's specified technology stack. It is structured to be fed directly into an AI coding assistant to generate the necessary backend and frontend code.

## Technical Specification: Invitation-Only Auth Flow

### 1. System Overview

**Objective**: Create a secure onboarding flow where users cannot sign up without a valid, admin-generated invitation.
**Mechanism**:
1.  Admin generates a magic link sent via Email or SMS.
2.  User clicks the link to validate the token.
3.  User authenticates via a Social Provider (OAuth), managed by the **Hono** proxy.
4.  The **NestJS** backend links the invite to the new user record in the **PostgreSQL** database.
5.  User is redirected to the Welcome Dashboard on the **Next.js** frontend.

### 2. Technology Stack

*   **Frontend**: Next.js
*   **Backend**: NestJS (TypeScript)
*   **Authentication Gateway**: Hono
*   **Database**: PostgreSQL with TypeORM
*   **Notification Service**: OneSignal (for Email and SMS)

### 3. Database Schema Requirements (PostgreSQL with TypeORM)

The AI should construct the database entities using **TypeORM** with the following specific fields:

**Table: `invitations`**
-   `id` (Primary Key, UUID)
-   `token` (String, Unique, Index): A secure, random hash (e.g., `crypto.randomBytes`).
-   `recipient_contact` (String): The email address or phone number.
-   `type` (Enum): `EMAIL` or `SMS`.
-   `status` (Enum): `PENDING`, `USED`, `EXPIRED`.
-   `expires_at` (Timestamp): Typically 24-48 hours.
-   `created_by` (Foreign Key): Links to the `users` table (Admin).

**Table: `users`**
-   `id` (Primary Key, UUID)
-   `auth_provider_id` (String): The ID returned from the social provider via Hono.
-   `email` (String, Unique)
-   `phone_number` (String, nullable)
-   `invitation_id` (Foreign Key, nullable): Links back to the `invitations` table.

### 4. Detailed Step-by-Step Logic

#### Phase 1: Invitation Generation (Admin Side on Next.js)

**Logic**:
1.  Admin, on a protected page in the **Next.js** app, inputs an email or phone number and clicks "Invite."
2.  The **Next.js** frontend sends a request to the **NestJS** backend.
3.  **NestJS Backend**:
    *   Generates a unique, secure token.
    *   Creates a record in the `invitations` table (PostgreSQL) with status `PENDING` using **TypeORM**.
4.  **Notification Service**:
    *   The **NestJS** service triggers **OneSignal** to send the notification.
    *   **Email/SMS Content**: The message will contain a magic link: `https://app.com/join?token={token}`.

#### Phase 2: User Landing & Validation (Next.js Frontend)

**Logic**:
1.  User clicks the link and lands on the **Next.js** application page `/join`.
2.  `useEffect` hook on page mount extracts the `token` from the URL query parameters.
3.  **API Call**: The frontend makes a `GET` request to the **NestJS** API: `/api/invitations/validate/{token}`.
4.  **If Valid**:
    *   The **NestJS** API returns a success response.
    *   The **Next.js** app stores the token in `localStorage` or `sessionStorage`.
    *   The UI displays "Sign in with Google/GitHub" buttons.
5.  **If Invalid/Expired**: The API returns an error, and the **Next.js** app redirects to an error page.

#### Phase 3: Social Authentication (Hono Gateway)

**Logic**:
1.  User clicks a "Sign in with..." button on the **Next.js** frontend.
2.  The frontend redirects the user to the **Hono** auth proxy's endpoint (e.g., `/auth/google`).
3.  **Hono Proxy**:
    *   Handles the entire OAuth 2.0 flow with the social provider.
    *   On successful authentication, it receives the user's profile (email, provider ID).
    *   It redirects the user back to the **Next.js** app's designated callback URL (e.g., `/auth/callback`).

#### Phase 4: Account Creation & Linking (NestJS Backend)

**Logic**: Triggered by the frontend after the redirect from Hono.
1.  The **Next.js** page at `/auth/callback` retrieves the invitation `token` from `localStorage`.
2.  It sends a request to the **NestJS** backend (e.g., `POST /api/auth/register`) with the `token` and the user details provided by Hono (which are now available to the frontend).
3.  **NestJS Backend - Final Security Check**:
    *   Verifies the `invitations` token is valid and `PENDING`.
    *   (Optional Strict Mode) Confirms the social auth email matches the `recipient_contact` in the invitation.
4.  **Execution (using TypeORM)**:
    *   Creates a new user in the `users` table.
    *   Updates the `invitations` table: sets `status` to `USED` and links the `user_id`.
    *   The **Hono** proxy, having authenticated the user, generates and returns a JWT/session token.
5.  **Redirect**: The **Next.js** frontend, upon receiving the success response and session token, redirects the user to `/dashboard/welcome`.

### 5. Edge Case Handling Rules

Instruct the AI to implement these scenarios within the **NestJS** backend:

**Scenario A: User already exists.**
-   **Action**: If a user with the social provider's email already exists in the `users` table, the **NestJS** backend should bypass the invitation logic, log the user in (via Hono's session management), and the frontend redirects to the main dashboard.

**Scenario B: Token Expired.**
-   **Action**: If the `GET /api/invitations/validate/{token}` call fails due to expiration, the **Next.js** frontend should display an option to "Request a new invitation." This action will trigger a notification to the original inviting Admin.

**Scenario C: Cross-Device Issue.**
-   **Action**: The magic link `token` should be a manageable length if manual entry is anticipated. The primary flow assumes the link is clicked directly.

