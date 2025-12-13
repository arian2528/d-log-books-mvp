# Core Business Logic Implementation Guide

This document provides a detailed, step-by-step guide for implementing the core business logic of the application. It translates the high-level concepts described in the [Core Business Value](./core-business-value.md) document into concrete frontend and backend development tasks.

---

## 1. Cascading Update Implementation

This feature ensures that when a log entry is signed (e.g., by a Pilot or Mechanic), the total times/cycles for the associated aircraft and its components are automatically updated.

### Frontend (Next.js)

-   **UI/UX Guidelines**: Before creating any components, developers must consult the [UI Preferences & Design System](./ui-preferences.md) document. All components should adhere to the specified colors, typography, spacing, and interaction patterns to ensure a consistent user experience.

-   **UI Components to be Built**:
    -   `LogEntryForm.tsx`: A form for creating new log entries (e.g., flight logs, maintenance logs). It will include fields for flight hours, cycles, etc.
    -   `SignLogEntryButton.tsx`: A button that triggers the signing action. This component will handle the API call and display loading/success/error states. It should use an optimistic UI update pattern.
    -   `AircraftStatusDashboard.tsx`: A dashboard component that displays the aircraft's total times. This component should re-fetch its data or have its cache invalidated after a successful signing operation.

-   **API Interaction**:
    -   The `SignLogEntryButton` will make a `POST` request to a new API endpoint (e.g., `/api/log-entries/:id/sign`).
    -   Data fetching for the dashboard will use SWR or React Query to handle caching and revalidation.

### Backend (NestJS)

-   **API Endpoints to be Created**:
    -   `POST /log-entries/:id/sign`:
        -   **Guard**: `JwtAuthGuard` to ensure the user is authenticated.
        -   **Authorization**: The service logic must verify that the user's role is permitted to sign this type of log entry (e.g., a Pilot can sign a flight log, a Mechanic can sign a maintenance log).
        -   **Validation**: Use a DTO with Zod to validate any incoming payload.
        -   **Business Logic**:
            1.  Find the `LogEntry` by its ID.
            2.  Mark the entry as signed and immutable.
            3.  Start a database transaction.
            4.  Fetch the associated `Aircraft` and its related components (e.g., `Engine`, `Propeller`).
            5.  Calculate the new total hours/cycles based on the log entry data.
            6.  Update the `total_hours` and `total_cycles` on the `Aircraft` and component records.
            7.  Commit the transaction.
            8.  Return a success response.

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

-   **Database Schema**:
    -   The `squawks` table must have a foreign key `aircraft_id` that links it to the `aircrafts` table.
    -   A `squawk_history` table should be created to log all status changes and transfers.
