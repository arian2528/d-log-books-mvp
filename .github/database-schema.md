
# Database Schema

This document outlines the database schema for the aviation record system, based on the entities defined in the Core Business Value document. It includes descriptions of core tables, junction tables for many-to-many relationships, and the data isolation strategy.

---

## 1. Data Isolation & Row-Level Security (RLS) Strategy

To ensure data integrity and enforce strict access control, the system implements PostgreSQL's Row-Level Security (RLS). This strategy is built on two core principles:

1.  **Control Over Actions (Signing)**: RLS policies restrict who can modify or sign log entries. The fundamental rule is that a user can only sign their own records. An attempt by one user to sign a log entry belonging to another will be blocked at the database level.

2.  **Control Over Visibility (Reviewing)**: RLS policies control who can view data based on their role and relationship to a specific aircraft. For example, an `Owner` or `Operator` can view all logs for aircraft in their fleet, while a `Pilot` can only see logs they have personally signed.

This dual approach guarantees that actions are strictly controlled at the individual level, while data visibility is managed through a flexible, role-based model.

---

## 2. Core Entities

### `aircraft`
This is the core table of the application, representing a physical aircraft. It is the central hub connecting to most other entities via the junction tables defined in Section 3.

**Columns:**
- `id` (PK, String): Unique identifier.
- `type` (String): Aircraft type.
- `registration_number` (String, Unique): The aircraft's registration number (e.g., "N12345").
- `model` (String): The specific model of the aircraft (e.g., "172 Skyhawk").
- `serial` (String): The manufacturer's serial number.
- `manufacturer` (String): The company that built the aircraft (e.g., "Cessna").
- `date_of_manufacture` (Date): Date of manufacture.
- `category` (Integer): Aircraft category.
- `hours` (Float): Total hours on the airframe.

### `engine`
Represents an aircraft engine, which can be tracked independently.

**Columns:**
- `id` (PK, String): Unique identifier.
- `aircraft_id` (FK, String): Foreign key to the `aircraft` table, indicating which aircraft it is currently installed on.
- `serial_number` (String, Unique): Engine serial number.
- `type` (String): Engine type.
- `manufacturer` (String): Engine manufacturer.
- `model` (String): Engine model.
- `date_mfg` (Date): Date of manufacture.
- `location` (String): Current location (e.g., installed on aircraft ID or in storage).
- `hours_since_overhaul` (Integer): Hours since last major overhaul.

### `propeller`
Represents an aircraft propeller, which can be tracked independently.

**Columns:**
- `id` (PK, String): Unique identifier.
- `aircraft_id` (FK, String): Foreign key to the `aircraft` table, indicating which aircraft it is currently installed on.
- `serial_number` (String, Unique): Propeller serial number.
- `type` (String): Propeller type.
- `manufacturer` (String): Propeller manufacturer.
- `model` (String): Propeller model.
- `date_mfg` (Date): Date of manufacture.
- `location` (String): Current location.
- `hours_since_overhaul` (Integer): Hours since last major overhaul.
- `hub_model` (String): Hub model number.
- `hub_serial` (String): Hub serial number.
- `blade_model` (String): Blade model number.
- `blade_serial` (String): Blade serial number.

### `compoonent_parts`
Represents other tracked components and parts.

**Columns:**
- `id` (PK, String): Unique identifier.
- `ecomponent` (String): Electronic component identifier.
- `part_name` (String): Name of the part.
- `mfg` (String): Manufacturer.
- `model` (String): Model number.
- `serial` (String): Serial number.
- `qty` (Integer): Quantity.
- `location` (String): Install location.
- `install_date` (Date): Date of installation.
- `exp_date` (Date): Expiration date (if applicable).
- `time_limit` (String): Time limit for use (e.g., hours, cycles).

---

## 3. People & Organization Entities

### `pilot`
Represents a pilot.

**Columns:**
- `id` (PK, String): Unique identifier.
- `name` (String): Pilot's full name.
- `license_number` (String): Pilot's license number.
- `ratings` (String): Pilot's ratings (e.g., "Instrument, Multi-Engine").
- `medical_certificate_expiry` (Date): Expiry date of medical certificate.

### `operator`
Represents the aircraft operator or company.

**Columns:**
- `id` (PK, String): Unique identifier.
- `name` (String): Operator's name.
- `certificate_number` (String): Operator's certificate number.
- `contact_info` (String): Contact information.

### `owner`
Represents the legal owner of an aircraft.

**Columns:**
- `id` (PK, String): Unique identifier.
- `name` (String): Owner's name.
- `contact_info` (String): Contact information.

### `mechanic`
Represents a certified maintenance professional.

**Columns:**
- `id` (PK, String): Unique identifier.
- `name` (String): Mechanic's full name.
- `license_number` (String): Mechanic's license number.
- `certificate_type` (String): Type of certification (e.g., "A&P", "IA").
- `certificate_expiry` (Date): Expiry date of certification.

---

## 4. Junction Tables (Many-to-Many Relationships)

### `aircraft_operators`
Links aircraft to the operators managing them.
- `aircraft_id` (FK, String): Foreign key to `aircraft.id`.
- `operator_id` (FK, String): Foreign key to `operator.id`.
- `assignment_date` (Date): Date the operator assignment began.
- `end_date` (Date, Nullable): Date the assignment ended.

### `aircraft_pilots`
Links aircraft to the pilots authorized to fly them.
- `aircraft_id` (FK, String): Foreign key to `aircraft.id`.
- `pilot_id` (FK, String): Foreign key to `pilot.id`.
- `authorization_date` (Date): Date the pilot was authorized.
- `end_date` (Date, Nullable): Date the authorization ended.

### `aircraft_owners`
Links aircraft to their owners.
- `aircraft_id` (FK, String): Foreign key to `aircraft.id`.
- `owner_id` (FK, String): Foreign key to `owner.id`.
- `ownership_percentage` (Decimal): The percentage of ownership.
- `start_date` (Date): Date ownership began.
- `end_date` (Date, Nullable): Date ownership ended.

### `aircraft_installed_parts`
Links aircraft to their installed components, supporting component modularity.
- `aircraft_id` (FK, String): Foreign key to `aircraft.id`.
- `component_part_id` (FK, String): Foreign key to `compoonent_parts.id`.
- `installation_date` (Date): Date the part was installed.
- `removal_date` (Date, Nullable): Date the part was removed.

### `aircraft_stc_alterations`
Links aircraft to any applied Supplemental Type Certificates or alterations.
- `aircraft_id` (FK, String): Foreign key to `aircraft.id`.
- `stc_alteration_id` (FK, String): Foreign key to `stc_alterations.id`.
- `application_date` (Date): Date the STC was applied.

---

## 5. Log & Record Entities

### `pilot_log`
A detailed log entry for a pilot's flight.

**Columns:**
- `id` (PK, String): Unique identifier for the log entry.
- `pilot_id` (FK, String): Foreign key to the `pilot` table.
- `aircraft_id` (FK, String): Foreign key to the `aircraft` table.
- `date` (Date): Date of the flight.
- `total_flight_time` (Decimal): Total duration of the flight.
- `departure_location` (String): Departure airport/location.
- `arrival_location` (String): Arrival airport/location.
- ... (and all other detailed fields from the DBML).

### `hardware_log`
A generic log for maintenance and work on physical components (airframe, engine, etc.).

**Columns:**
- `id` (PK, String): Unique identifier.
- `mechanic_responsible_id` (FK, String): Foreign key to the `mechanic` table.
- `log_type` (String): Type of log (e.g., "Engine", "Airframe").
- `flight_hours` (Float): Aircraft hours at the time of the log entry.
- `date` (Date): Date of the work.
- `work_type` (String): Type of work performed (e.g., "Inspection", "Repair").
- `work_order` (String): Work order number.
- `description` (String): Description of the work.
- `component` (String): Specific component worked on.

### `operator_log`
A log for operational records.

**Columns:**
- `id` (PK, String): Unique identifier.
- `operator_responsible_id` (FK, String): Foreign key to the `operator` table.
- `flight_hours` (Float): Aircraft hours at the time of the log entry.
- `date` (Date): Date of the operation.
- `operation_type` (String): Type of operation.
- `work_order` (String): Associated work order.
- `description` (String): Description of the operation.
- `other_data` (String): Additional data.

---

## 6. Compliance & Certification Entities

### `certificate`
Represents a pilot's certificate.

**Columns:**
- `id` (PK, String): Unique identifier.
- `pilot_id` (FK, String): Foreign key to the `pilot` table.
- `grade` (String): Certificate grade (e.g., "Private", "Commercial").
- `number` (String): Certificate number.
- `date_issue` (Date): Date of issue.

### `rating`
Represents a rating on a pilot's certificate.

**Columns:**
- `id` (PK, String): Unique identifier.
- `pilot_id` (FK, String): Foreign key to the `pilot` table.
- `category_class_or_type` (String): The specific rating (e.g., "AMEL", "B737").
- `date_issue` (Date): Date of issue.

### `medical_flight_proficiency`
Tracks a pilot's medical and proficiency status.

**Columns:**
- `id` (PK, String): Unique identifier.
- `pilot_id` (FK, String): Foreign key to the `pilot` table.
- `medical_certificate_date` (Date): Date of medical certificate issuance.
- `medical_certificate_class` (String): Class of medical certificate.
- `flight_review_date` (Date): Date of last flight review.
- `instrument_proficiency_date` (Date): Date of last instrument proficiency check.

### `stc_alterations`
Represents a Supplemental Type Certificate or major alteration.

**Columns:**
- `id` (PK, String): Unique identifier.
- `stc_number` (String): The STC number.
- `mfg` (String): Manufacturer associated with the STC.
- `model` (String): Model associated with the STC.
- `serial_number` (String): Serial number of the altered part.
- `install_date` (Date): Date of installation.
- `location` (String): Location of the alteration.

### `directives_bulletines`
Represents Airworthiness Directives (ADs) and Service Bulletins (SBs).

**Columns:**
- `id` (PK, String): Unique identifier.
- `type` (String): "AD" or "SB".
- `number` (String): The AD/SB number.
- `hours` (Float): Compliance hours.
- `description` (String): Description of the directive.
- `action` (String): Required action.
- `compliance_date` (Date): Date of compliance.
- `compliance_data` (String): Details of compliance.
- `made_by` (String): Who performed the compliance.

### `aircraft_hours_entries`
Represents a log of aircraft hours at specific maintenance events.

**Columns:**
- `id` (PK, String): Unique identifier.
- `aircraft_id` (FK, String): Foreign key to the `aircraft` table.
- `hours` (Float): Total airframe hours at the time of entry.
- `date` (Date): Date of the entry.


