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

## 3. Target Audience

This application is designed for **aircraft owners, operators, pilots, and certified maintenance professionals (A&P, IA)** who require a reliable, compliant, and efficient system to manage the complex web of aviation records.

## The Central `Aircraft` Entity

The `Aircraft` entity is the core of the entire system. It acts as the central hub to which all other major data models are connected, representing a specific, physical aircraft. Its relationships define who can operate it, who owns it, what parts it's made of, and what regulations apply to it.

### Key Relationships (Many-to-Many)

The `Aircraft` entity maintains many-to-many relationships with several other key entities, allowing for a flexible and scalable data structure:

-   **`Aircraft` <> `Aircraft Parts`**: An aircraft is composed of many parts, and a single part type (e.g., a specific model of tire) can be used on many different aircraft. This relationship tracks the specific components installed on an aircraft.

-   **`Aircraft` <> `Operators`**: An aircraft can be managed by multiple operators over its lifetime (or even concurrently in some arrangements), and an operator can manage a fleet of multiple aircraft.

-   **`Aircraft` <> `Pilots`**: A pilot is authorized to fly multiple aircraft, and a single aircraft can be flown by many different pilots.

-   **`Aircraft` <> `Owners`**: An aircraft can have multiple co-owners, and an individual or entity can own multiple aircraft.

-   **`Aircraft` <> `Directives/Bulletins`**: A single airworthiness directive or service bulletin can apply to many aircraft, and a single aircraft may be subject to numerous directives and bulletins.

-   **`Aircraft` <> `STC/Alterations`**: A Supplemental Type Certificate (STC) or alteration can be applied to many aircraft, and one aircraft can have multiple STCs or alterations applied to it.
