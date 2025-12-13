# Core Business Value: The "All-In-One" Aviation Record System

This document outlines the core business value and philosophy for the unified aviation record system.

## 1. System Overview & Primary Goal

**Objective**: To build a unified platform that acts as the "Single Source of Truth" for an aircraft's entire history by integrating five distinct but interconnected logbooks:

*   **Pilot Log**: For individual pilots to record personal flight time, landings, and currency. The pilot's flight entry is the primary trigger for the cascading update.
*   **Operator Log**: For the aircraft owner or company (the "Operator") to manage operational data, such as crew assignments, dispatch records, and maintenance scheduling.
*   **Aircraft (Airframe) Log**: Total Time in Service, Inspections, Airworthiness Directives (ADs).
*   **Engine Log**: Time Since New, Time Since Overhaul, oil changes.
*   **Propeller Log**: Time Since New, Time Since Overhaul.
*   **Avionics Log**: Software updates, VOR checks, transponder certifications.

## 2. Core Values & Key Differentiators

Our value proposition is defined by a unique architecture designed for accuracy, modularity, and integrity.

### Value 1: The "Cascading Update" Architecture
**Concept**: Users should never enter data twice. A single flight entry propagates mathematically correct updates across the entire system.

**User Benefit**: When a pilot logs a 2.0-hour flight, the system automatically adds 2.0 hours to the pilot's total time, the airframe's total time, the engine's time since overhaul, and the propeller's time since new. This eliminates redundant data entry, prevents mathematical errors, and ensures all records are perfectly synchronized.

### Value 2: Component Independence (Modularity)
**Concept**: Aircraft parts are swappable assets. The logbook must follow the component, not just the aircraft it's attached to.

**User Benefit**: An engine can be removed from one aircraft and installed on another, and its entire maintenance history (Time Since Overhaul, repairs, etc.) travels with it. This provides a complete, unbroken, and legally compliant history for every major component, increasing asset value and safety.

### Value 3: Role-Based Integrity & Immutability
**Concept**: Different logbook entries require different levels of authority to "sign off."

**User Benefit**: A pilot can sign their own flight log, but only a certified A&P Mechanic or Inspector-Authorized (IA) mechanic can sign off on an annual inspection or major repair. Once a maintenance entry is signed, it becomes **immutable** (read-only and locked), preserving the legal chain of custody and guaranteeing the integrity of the aircraft's records.

---

## 3. Key Feature Implementations

Building on our core values, the platform delivers the following key features:

### For Pilots:
-   **Auto-Logging**: The system uses the device's GPS to automatically detect takeoff and landing times, pre-filling flight log entries to ensure accuracy and reduce manual data entry.
-   **Squawk Management**: Pilots can instantly report maintenance issues ("squawks") directly from the app, complete with photos and notes. These squawks immediately appear in the maintenance queue for mechanics to address.

### For Maintenance Professionals (A&P, IA):
-   **Digital Work Orders**: Create, assign, and manage maintenance tasks and work orders. Mechanics can document their work, and Inspector-Authorized (IA) personnel can provide digital sign-offs, locking the record to ensure immutability.
-   **AD/SB Monitoring**: The system automatically tracks Airworthiness Directives (ADs) and Service Bulletins (SBs) applicable to an aircraft and its specific components based on serial numbers. It provides automated alerts for upcoming or overdue compliance tasks.

### General Platform Features:
-   **Digital Signatures**: All legally significant entries (e.g., flight logs, maintenance sign-offs, inspections) are signed digitally, creating a secure, compliant, and auditable chain of custody that meets regulatory standards.
-   **Paper Log Digitization (OCR)**: A core feature is the ability to scan and digitize historical paper logs. Using Optical Character Recognition (OCR), the system makes these scanned records fully searchable, integrating the aircraft's entire paper history into the digital platform.

---

## 4. Target Audience

This application is designed for **aircraft owners, operators, pilots, and certified maintenance professionals (A&P, IA)** who require a reliable, compliant, and efficient system to manage the complex web of aviation records.

## 5. The Central `Aircraft` Entity

The `Aircraft` entity is the core of the entire system. It acts as the central hub to which all other major data models are connected, representing a specific, physical aircraft. Its relationships define who can operate it, who owns it, what parts it's made of, and what regulations apply to it.

### Key Relationships (Many-to-Many)

The `Aircraft` entity maintains many-to-many relationships with several other key entities, allowing for a flexible and scalable data structure:

-   **`Aircraft` <> `Aircraft Parts`**: An aircraft is composed of many parts, and a single part type (e.g., a specific model of tire) can be used on many different aircraft. This relationship tracks the specific components installed on an aircraft.

-   **`Aircraft` <> `Operators`**: An aircraft can be managed by multiple operators over its lifetime (or even concurrently in some arrangements), and an operator can manage a fleet of multiple aircraft.

-   **`Aircraft` <> `Pilots`**: A pilot is authorized to fly multiple aircraft, and a single aircraft can be flown by many different pilots.

-   **`Aircraft` <> `Owners`**: An aircraft can have multiple co-owners, and an individual or entity can own multiple aircraft.

-   **`Aircraft` <> `Directives/Bulletins`**: A single airworthiness directive or service bulletin can be applied to many aircraft, and a single aircraft may be subject to numerous directives and bulletins.

-   **`Aircraft` <> `STC/Alterations`**: A Supplemental Type Certificate (STC) or alteration can be applied to many aircraft, and one aircraft can have multiple STCs or alterations applied to it.

---

## 6. Detailed Entity Properties

This section provides a detailed breakdown of each entity's properties using the Database Markup Language (DBML).

```dbml
// FAA Compliance Logs Data Model

title "FAA Compliance Logs Data Model"

// define tables
Table aircraft [icon: airplay, color: blue]{
  id string [pk]
  type string
  registration_number string
  model string
  serial string
  manufacturer string
  date_of_manufacture date
  category int
  hours float
}

Table engine [icon: cpu, color: orange]{
  id string [pk]
  serial_number string
  type string
  manufacturer string
  model string
  serial string
  date_mfg date
  location string
  hours_since_overhaul int
}

Table propeller [icon: wind, color: green]{
  id string [pk]
  serial_number string
  type string
  manufacturer string
  model string
  date_mfg string
  location string
  hours_since_overhaul int
  hub_model string
  hub_serial string
  blade_model string
  blade_serial string
}

Table compoonent_parts [icon: radio, color: purple]{
  id string [pk]
  ecomponent string
  part_name string
  mfg string
  model string
  serial string
  qty int
  location string
  install_date date
  exp_date date
  time_limit string
}

Table operator [icon: user-check, color: yellow]{
  id string [pk]
  name string
  certificate_number string
  contact_info string
}

Table pilot [icon: user, color: red]{
  id string [pk]
  name string
  license_number string
  ratings string
  medical_certificate_expiry date
}
Table pilot_log [icon: book-open, color: lightblue] {
  
  id string [pk]
  date date
  total_flight_time decimal
  departure_location string
  arrival_location string
  simulator_location string
  aircraft_id string
  aircraft_type string
  safety_pilot_name string
  solo_time decimal
  pic_time decimal
  sic_time decimal
  training_time decimal
  day_time decimal
  night_time decimal
  actual_instrument_time decimal
  simulated_instrument_time decimal
  nvg_time decimal
  instructor_endorsement_description string
  instructor_signature string
  instructor_certificate_number string
  instructor_certificate_expiry date
  instrument_approach_location string
  instrument_approach_type string
  route_of_flight string
  takeoffs_day int
  takeoffs_night int
  landings_day int
  landings_night int
  cross_country_time decimal
  complex_time decimal
  high_performance_time decimal
  tailwheel_time decimal
  notes string
  pilot_id string
}
Table certificate [icon: award, color: gold] {
  
  id string [pk]
  grade string
  number string
  date_issue date
}
Table rating [icon: star, color: pink] {
  
  id string [pk]
  category_class_or_type string
  date_issue date
}
Table medical_flight_proficiency [icon: activity, color: teal] {
  
  id string [pk]
  pilot_id string
  medical_certificate_date date
  medical_certificate_class string
  flight_review_date date
  instrument_proficiency_date date
}
Table aircraft_hours_mentries [icon: clock, color: gray] {
  
  id string [pk]
  hours float
  date date
  aircraft_id string
}
Table stc_alterations [icon: tool, color: brown] {
  
  id string [pk]
  stc_number string
  mfg string
  model string
  serial_number string
  install_date date
  location string
}
Table directives_bulletines [icon: alert-triangle, color: magenta] {
  
  id string [pk]
  type string
  number date
  hours float
  description string
  action string
  compliance_date date
  compliance_data string
  made_by string
}
Table hardware_log [icon: settings, color: gray] {
  
  id string [pk]
  log_type string
  flight_hours float
  date date
  work_type string
  work_order string
  description string
  mechanic_responsible string
  component string
}
Table operator_log [icon: file, color: yellow] {
  
  id string [pk]
  flight_hours float
  date date
  operation_type string
  work_order string
  description string
  operator_responsible string
  other_data string
}
```

