# Core Business Logic Implementation Guide

This document provides a detailed, step-by-step guide for implementing the core business logic of the application. It translates the high-level concepts described in the [Core Business Value](./core-business-value.md) document into concrete frontend and backend development tasks.

---

## 1. Cascading Update Implementation

This feature ensures that when a log entry is signed (e.g., by a Pilot or Mechanic), the total times/cycles for the associated aircraft and its components are automatically updated.

### Frontend (Next.js)

-   **UI/UX Guidelines**: Before creating any components, developers must consult the [UI Preferences & Design System](./ui-preferences.md) document. All components should adhere to the specified colors, typography, spacing, and interaction patterns to ensure a consistent user experience.

-   **UI Components to be Built**:
    -   `LogEntryForm.tsx`: A dynamic form component for creating new log entries. This component's primary responsibility is to render the correct set of input fields based on the user's role and the type of log being created.
        -   **Role-Based Logic**: The component will determine the user's role (e.g., `Pilot`, `Mechanic`) from the application's authentication context.
        -   **Dynamic Field Rendering**: Based on the role, it will display fields corresponding to the appropriate log entity as defined in the `core-business-value.md` document.
            -   **Example (Pilot)**: If the user has the `Pilot` role, the form will render fields for the `pilot_log` entity (e.g., `total_flight_time`, `departure_location`, `landings_day`).
            -   **Example (Mechanic)**: If the user has the `Mechanic` role, the form will render fields for the `hardware_log` entity (e.g., `log_type`, `work_type`, `description`, `component`).
    -   `SignLogEntryButton.tsx`: A button that triggers the signing action. This component must be idempotent. It will handle the API call, display loading/success/error states, and should disable itself immediately on click to prevent multiple submissions. It should also use an optimistic UI update pattern.
    -   `AircraftStatusDashboard.tsx`: A dashboard component that displays the aircraft's total times. This component should re-fetch its data or have its cache invalidated after a successful signing operation.

-   **API Interaction**:
    -   The `SignLogEntryButton` will make a `POST` request to a new API endpoint (e.g., `/api/log-entries/:id/sign`).
    -   Data fetching for the dashboard will use SWR or React Query to handle caching and revalidation.

-   **Testing**:
    -   **Component Tests**: Write tests for `LogEntryForm.tsx` and `SignLogEntryButton.tsx` using React Testing Library to verify form handling, user input, and correct API call invocation.
    -   **Integration Test**: Create a test that simulates the entire signing flow, mocking the API response to verify the UI updates correctly (e.g., dashboard data is refetched).

### Backend (NestJS)

-   **API Endpoints to be Created**:
    -   `POST /log-entries/:id/sign`:
        -   **Idempotency**: This endpoint must be idempotent. If the same request is received multiple times, it should not result in duplicate updates. The business logic should first check if the log entry has already been signed. If it has, the endpoint should return the existing signed data without performing the update again.
        -   **Guard**: `JwtAuthGuard` to ensure the user is authenticated.
        -   **Authorization**: The service logic must verify that the user's role is permitted to sign this type of log entry (e.g., a Pilot can sign a flight log, a Mechanic can sign a maintenance log).
        -   **Validation**: Use a DTO with Zod to validate any incoming payload.
        -   **Business Logic**:
            1.  Find the `LogEntry` by its ID.
            2.  **Check if already signed**. If `signed_at` is not null, return a success response immediately.
            3.  Mark the entry as signed and immutable.
            4.  Start a database transaction.
            5.  Fetch the associated `Aircraft` and its related components (e.g., `Engine`, `Propeller`).
            6.  Calculate the new total hours/cycles based on the log entry data.
            7.  Update the `total_hours` and `total_cycles` on the `Aircraft` and component records.
            8.  Commit the transaction.
            9.  Return a success response.

-   **Testing**:
    -   **Unit Test**: Write a unit test for the service method containing the business logic. Mock the database repository to test the calculation and transaction logic in isolation.
    -   **Integration Test**: Write an integration test for the `POST /log-entries/:id/sign` endpoint using Supertest. Use a test database to verify that the endpoint correctly authenticates the user, validates the payload, and updates the database records as expected.

-   **Database Schema**:
    -   The `log_entries` table will need a `signed_at` timestamp and a `signed_by_user_id` foreign key.
    -   The `aircrafts`, `engines`, and `propellers` tables will need `total_hours` and `total_cycles` columns.

---

## 2. Component Independence & Squawk Management

This feature allows maintenance discrepancies ("squawks") to be managed independently of the aircraft and transferred between aircraft.

### Frontend (Next.js)

-   **UI/UX Guidelines**: Before creating any components, developers must consult the [UI Preferences & Design System](./ui-preferences.md) document. All components should adhere to the specified colors, typography, spacing, and interaction patterns to ensure a consistent user experience.

-   **UI Components to be Built**:
    -   `SquawkList.tsx`: A component that lists all open squawks for a given aircraft.
    -   `SquawkItem.tsx`: Displays a single squawk with its status and provides an action to clear or defer it.
    -   `CreateSquawkForm.tsx`: A form for creating a new squawk.
    -   `TransferSquawkModal.tsx`: A modal UI that allows an authorized user (e.g., Mechanic) to select a squawk and transfer it to another aircraft in the system.

-   **API Interaction**:
    -   `POST /squawks` to create a new squawk.
    -   `PATCH /squawks/:id` to update a squawk's status (e.g., clear, defer).
    -   `POST /squawks/:id/transfer` to move a squawk to a different aircraft.

-   **Testing**:
    -   **Component Tests**: Write tests for `CreateSquawkForm.tsx` and `TransferSquawkModal.tsx` to ensure they function correctly.
    -   **E2E Test**: Create an end-to-end test using Cypress or Playwright that simulates a user creating a squawk, and then a different user (e.g., a mechanic) transferring it to another aircraft.

### Backend (NestJS)

-   **API Endpoints to be Created**:
    -   `POST /squawks`: Creates a new squawk associated with an aircraft.
    -   `PATCH /squawks/:id`: Updates the status of a squawk. Must include authorization logic to ensure only permitted roles (e.g., Mechanic, IA) can clear squawks.
    -   `POST /squawks/:id/transfer`:
        -   **Validation**: DTO must include the `target_aircraft_id`.
        -   **Business Logic**:
            1.  Find the `Squawk` by its ID.
            2.  Verify the squawk is in a "transferable" state (e.g., not cleared).
            3.  Update the `aircraft_id` on the squawk record to the `target_aircraft_id`.
            4.  Create a historical record of the transfer for audit purposes.

-   **Testing**:
    -   **Unit Tests**: Write unit tests for the service logic, particularly for the transfer functionality, to ensure it correctly handles state verification and historical record creation.
    -   **Integration Tests**: Write integration tests for all three squawk-related endpoints (`/squawks`, `/squawks/:id`, `/squawks/:id/transfer`) to validate the full request/response cycle and database interactions.

-   **Database Schema**:
    -   The `squawks` table must have a foreign key `aircraft_id` that links it to the `aircrafts` table.
    -   A `squawk_history` table should be created to log all status changes and transfers.

---

## 3. Data Isolation & Row-Level Security (RLS) Implementation

This feature implements PostgreSQL Row-Level Security (RLS) to enforce strict data isolation rules. The security model controls both who can sign logs (actions) and who can view log data (visibility).

### Backend (NestJS + PostgreSQL)

-   **Database Schema Updates**:
    -   Add new core entities:
        -   `mechanic` table with fields: `id`, `name`, `license_number`, `certificate_type`, `certificate_expiry`.
        -   `owner` table with fields: `id`, `name`, `contact_info`.
    -   Add junction tables to support many-to-many relationships:
        -   `aircraft_operators`: Links aircraft to operators with `aircraft_id`, `operator_id`, `assignment_date`, `end_date`. Composite primary key on `(aircraft_id, operator_id)`.
        -   `aircraft_pilots`: Links aircraft to authorized pilots with `aircraft_id`, `pilot_id`, `authorization_date`, `end_date`. Composite primary key on `(aircraft_id, pilot_id)`.
        -   `aircraft_owners`: Links aircraft to owners with `aircraft_id`, `owner_id`, `ownership_percentage`, `start_date`, `end_date`. Composite primary key on `(aircraft_id, owner_id)`.
        -   `aircraft_installed_parts`: Tracks component installation with `aircraft_id`, `component_part_id`, `installation_date`, `removal_date`. Composite primary key on `(aircraft_id, component_part_id)`.
        -   `aircraft_stc_alterations`: Links STCs to aircraft with `aircraft_id`, `stc_alteration_id`, `application_date`. Composite primary key on `(aircraft_id, stc_alteration_id)`.

-   **RLS Policy Implementation**:
    -   **Control Over Actions (Signing)**:
        -   Create RLS policies on `pilot_log`, `hardware_log`, and `operator_log` tables that restrict `UPDATE` and `INSERT` operations.
        -   Policy Rule: A user can only sign (update the `signed_at` and `signed_by_user_id` fields) a log entry if the `signed_by_user_id` matches their authenticated user ID.
        -   Example SQL Policy for `pilot_log`:
          ```sql
          CREATE POLICY pilot_sign_own_logs ON pilot_log
            FOR UPDATE
            USING (signed_by_user_id = current_setting('app.current_user_id')::uuid);
          ```
    -   **Control Over Visibility (Reviewing)**:
        -   Create RLS policies on log tables that grant `SELECT` access based on the user's role and their relationship to the aircraft.
        -   Policy Rules:
            -   A `Pilot` can see all log entries they personally signed for any aircraft they have flown.
            -   An `Operator` can see all log entries for any aircraft in their fleet (via the `aircraft_operators` junction table).
            -   An `Owner` can see all log entries for any aircraft they own (via the `aircraft_owners` junction table).
            -   A `Mechanic` can see maintenance-related logs (`hardware_log`) for aircraft they are assigned to work on.
        -   Example SQL Policy for `pilot_log` (Pilot visibility):
          ```sql
          CREATE POLICY pilot_view_own_logs ON pilot_log
            FOR SELECT
            USING (pilot_id = current_setting('app.current_user_id')::uuid);
          ```
        -   Example SQL Policy for `pilot_log` (Operator/Owner visibility):
          ```sql
          CREATE POLICY operator_view_fleet_logs ON pilot_log
            FOR SELECT
            USING (
              aircraft_id IN (
                SELECT aircraft_id FROM aircraft_operators
                WHERE operator_id = current_setting('app.current_user_id')::uuid
                AND (end_date IS NULL OR end_date > NOW())
              )
            );
          ```

-   **Application-Level Context**:
    -   In the NestJS application, create a middleware or interceptor that sets the PostgreSQL session variable `app.current_user_id` at the start of each request based on the authenticated JWT token.
    -   Example:
      ```typescript
      await this.dataSource.query(
        `SET LOCAL app.current_user_id = $1`,
        [user.id]
      );
      ```

-   **TypeORM Entity Updates**:
    -   Update the `Aircraft` entity to include relationships to the new junction tables:
        -   `@ManyToMany(() => Operator, { through: () => AircraftOperators })`
        -   `@ManyToMany(() => Pilot, { through: () => AircraftPilots })`
        -   `@ManyToMany(() => Owner, { through: () => AircraftOwners })`
        -   `@ManyToMany(() => ComponentPart, { through: () => AircraftInstalledParts })`
        -   `@ManyToMany(() => StcAlteration, { through: () => AircraftStcAlterations })`
    -   Create new TypeORM entities for `Mechanic`, `Owner`, and all junction tables.

-   **Testing**:
    -   **Unit Tests**: Write unit tests for the RLS policy logic by mocking different user roles and verifying that the correct session variables are set.
    -   **Integration Tests**: Write integration tests that authenticate as different user types (Pilot, Operator, Owner, Mechanic) and verify:
        -   A user cannot sign a log entry that doesn't belong to them (should return a 403 Forbidden).
        -   A user can only see log entries they are authorized to view based on their role and aircraft associations.
    -   **E2E Tests**: Create end-to-end tests that simulate realistic scenarios:
        -   A Pilot signs their own log and verifies the signature is recorded.
        -   An Operator views all logs for their fleet.
        -   A Mechanic attempts to view a pilot log for an aircraft they are not assigned to and is denied access.
