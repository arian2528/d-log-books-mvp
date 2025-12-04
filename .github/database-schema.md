
# Database Schema Instructions

This document outlines the database schema for the aviation record system. The central entity is the `Aircraft`, which is connected to various other entities through many-to-many and one-to-one relationships.

## 1. Central Entity: `aircrafts`

This is the core table of the application.

**Columns:**
- `id` (Primary Key, UUID): Unique identifier for the aircraft.
- `tailNumber` (String, Unique): The aircraft's registration number (e.g., "N12345").
- `manufacturer` (String): The company that built the aircraft (e.g., "Cessna").
- `model` (String): The specific model of the aircraft (e.g., "172 Skyhawk").
- `serialNumber` (String): The manufacturer's serial number.

## 2. Entities with Many-to-Many Relationship to `aircrafts`

These entities can be associated with multiple aircraft, and each aircraft can be associated with multiple instances of these entities. A join table should be used for each relationship.

### `pilots`
- `id` (Primary Key, UUID)
- `name` (String)
- **Relationship**: `aircrafts_pilots` join table (`aircraft_id`, `pilot_id`).

### `owners`
- `id` (Primary Key, UUID)
- `name` (String)
- **Relationship**: `aircrafts_owners` join table (`aircraft_id`, `owner_id`).

### `operators`
- `id` (Primary Key, UUID)
- `name` (String)
- **Relationship**: `aircrafts_operators` join table (`aircraft_id`, `operator_id`).

### `aircraft_parts`
- `id` (Primary Key, UUID)
- `partName` (String)
- `partNumber` (String)
- **Relationship**: `aircrafts_parts` join table (`aircraft_id`, `part_id`).

### `stc_alterations` (Supplemental Type Certificate)
- `id` (Primary Key, UUID)
- `description` (Text)
- **Relationship**: `aircrafts_stc_alterations` join table (`aircraft_id`, `stc_alteration_id`).

### `directives_bulletins`
- `id` (Primary Key, UUID)
- `title` (String)
- `description` (Text)
- **Relationship**: `aircrafts_directives_bulletins` join table (`aircraft_id`, `directive_bulletin_id`).

## 3. Logbook Entities with One-to-One Relationship to `aircrafts`

Each aircraft has its own set of logbooks. This is a one-to-one relationship where the logbook's existence is tied to a specific aircraft.

### `pilot_logs`
- `id` (Primary Key, UUID)
- `aircraft_id` (Foreign Key to `aircrafts.id`)

### `operator_logs`
- `id` (Primary Key, UUID)
- `aircraft_id` (Foreign Key to `aircrafts.id`)

### `airframe_logs`
- `id` (Primary Key, UUID)
- `aircraft_id` (Foreign Key to `aircrafts.id`)

### `engine_logs`
- `id` (Primary Key, UUID)
- `aircraft_id` (Foreign Key to `aircrafts.id`)

### `propeller_logs`
- `id` (Primary Key, UUID)
- `aircraft_id` (Foreign Key to `aircrafts.id`)

### `avionics_logs`
- `id` (Primary Key, UUID)
- `aircraft_id` (Foreign Key to `aircrafts.id`)

