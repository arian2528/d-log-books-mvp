# Agnostic ETL Process Strategy

This document outlines a standardized, provider-agnostic strategy for Extract, Transform, and Load (ETL) processes. The goal is to create a flexible framework that can be adapted for various data integration needs, such as initial data loading, periodic data synchronization from external sources, or processing user-uploaded bulk data.

---

## Core Principles

All ETL processes, regardless of their specific purpose, must adhere to the following principles:

1.  **Idempotency**: Rerunning an ETL job with the same input data must not cause duplicate records or incorrect data. The system's state should be the same after one run or multiple runs of the same job.
2.  **Scalability**: The process must be designed to handle growth in data volume without significant degradation in performance. This often involves processing data in chunks or streams rather than all at once.
3.  **Observability**: Every ETL job must be thoroughly logged. It should be easy to determine when a job ran, what its outcome was (success, failure, partial success), how many records were processed, and what errors occurred.
4.  **Data Quality & Validation**: Data should be validated against a defined schema at the earliest possible stage. Invalid data should either be rejected and logged or cleaned/quarantined according to predefined business rules.
5.  **Transactional Loading**: The final "Load" step should be transactional to prevent partial data updates. If any part of the load fails, the entire transaction should be rolled back to maintain data integrity.

---

## The Three Stages of ETL

### 1. Extract

This stage is concerned with acquiring raw data from its source.

-   **Data Sources**:
    -   **Files**: CSV, JSON, XML, or Excel files uploaded by users or placed in a designated storage location (e.g., AWS S3, Google Cloud Storage).
    -   **APIs**: Data fetched from external third-party REST or GraphQL APIs.
    -   **Databases**: Data pulled from other databases, either via a direct connection or from a database dump.

-   **Extraction Strategy**:
    -   **Full Load**: The entire dataset is extracted. Best for small datasets or initial, one-time loads.
    -   **Incremental Load**: Only new or modified data since the last run is extracted. This is typically managed using a "high-water mark" (e.g., the latest `updated_at` timestamp or `id` from the last run).

-   **Staging Area**:
    -   It is highly recommended to land the raw, unaltered data in a temporary staging area before transformation. This could be a dedicated folder in cloud storage or a separate schema in the database (`staging`). This practice aids in debugging and allows transformations to be re-run without re-extracting the data.

### 2. Transform

This is where the raw data is converted into the format required by the target system. This stage contains the core business logic of the ETL process.

-   **Common Transformations**:
    -   **Cleaning**: Handling missing values, removing duplicate records, correcting typos.
    -   **Standardizing**: Formatting dates, phone numbers, or addresses into a consistent structure.
    -   **Enriching**: Combining the source data with data from other systems (e.g., joining user data with role information).
    -   **Restructuring**: Pivoting, unpivoting, or aggregating data to match the target database schema.
    -   **Validation**: Rigorously checking data against business rules and the target schema using tools like **Zod**.

### 3. Load

This final stage involves inserting the transformed data into the target database (PostgreSQL).

-   **Loading Strategy**:
    -   **Insert**: For new data only.
    -   **Update**: For modifying existing records.
    -   **Upsert (Update or Insert)**: The most common and robust strategy for idempotent loads. In PostgreSQL, this is efficiently handled with the `ON CONFLICT DO UPDATE` statement.

-   **Implementation**:
    -   The load should be performed within a database transaction.
    -   For large datasets, data should be loaded in batches (e.g., 1,000 records at a time) to manage memory usage and transaction log size. Each batch should be its own transaction.

---

## Orchestration and Technology

-   **Orchestration**: This refers to how ETL jobs are scheduled and triggered.
    -   **Manual**: A job is triggered manually by a developer or via an internal admin panel.
    -   **Scheduled**: A job runs on a recurring schedule (e.g., every night at 2 AM). This can be managed with a simple `cron` job or a scheduled task in a cloud environment.
    -   **Event-Driven**: A job is triggered by an event, such as a file being uploaded to a specific S3 bucket.
    -   **Recommended Tooling**: For simple, scheduled tasks, **GitHub Actions** is a sufficient and well-integrated orchestrator. For complex workflows with many dependencies, a dedicated tool like **Airflow** or **Prefect** would be more appropriate post-MVP.

-   **Technology Stack (Agnostic Recommendations)**:
    -   **Scripting**:
        -   **Python**: The de-facto standard for ETL scripting, with powerful libraries like **Pandas** or **Polars** for data manipulation.
        -   **Golang**: An excellent choice for high-performance, concurrent ETL jobs, especially when dealing with API-based extraction.
    -   **Containerization**: All ETL logic should be packaged in a **Docker** container. This ensures the runtime environment is consistent and makes the job portable and easy to run on any system (local machine, CI/CD runner, cloud service).

---

## Example MVP Implementation: CSV Upload

1.  **Extract**: A user uploads a CSV file of aircraft maintenance records via a secure endpoint in the NestJS application. The application saves this raw file to a designated folder in Google Cloud Storage.
2.  **Transform & Load (Trigger)**: The file upload event triggers a Google Cloud Run job. This job runs a containerized **Python script**.
3.  **Transform (In Script)**:
    -   The script downloads the CSV file from storage.
    -   It uses **Pandas** to read the data into a DataFrame.
    -   It uses **Zod** (or a Python equivalent like Pydantic) to validate each row. Invalid rows are logged to a separate error file.
    -   It transforms the valid data to match the PostgreSQL `maintenance_logs` table schema.
4.  **Load (In Script)**:
    -   The script connects to the PostgreSQL database.
    -   It iterates through the transformed data in batches of 500.
    -   For each batch, it begins a transaction and uses an `UPSERT` command to load the data, preventing duplicates.
5.  **Observability**: The script logs its progress, final status (success/failure), and the count of processed/failed records to `stdout`, which is captured by Google Cloud Logging.
