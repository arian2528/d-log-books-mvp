SaaS MVP Development Instructions (instructions.md)

1. Project Goal & MVP Definition

Goal: To launch a Minimum Viable Product (MVP) for a role-based SaaS application designed to

$$**Placeholder: Describe the Core Business Value/Action Here**$$

.

MVP Definition (Key Differentiator): The MVP must enable users to perform the core value proposition and demonstrate the robust, role-based access control system.

2. Core MVP Features

The scope is strictly limited to the following features to ensure rapid deployment:

A. User Management & Authentication

Social Sign-In: Users will authenticate exclusively through a supported Social Provider (e.g., GitHub, Google) managed by the Hono proxy. No email/password flow.

Role Assignment: Initial users are assigned one of two roles: Admin or Standard User.

User Profile: Simple view/edit of name and email.

B. Core Data Functionality (The "Do" Feature)

Create/Read Entity: Users can create and view Entity records (e.g., Projects, Tasks, or Records).

Role-Based Access Control (RBAC) Logic:

Standard User: Can only read/write their own Entity records.

Admin: Can read/write/delete any Entity record.

C. Basic Dashboard

A landing page that displays a list of accessible Entity records based on the user's assigned role.

3. Technical Stack & Architecture

Component

Technology

Role

Frontend

Next.js

Presentation layer, static/server-side generation of role-specific pages.

Auth Proxy

Hono better-auth-proxy

Secure authentication gatekeeper and token validation layer.

Monolith BE

NestJS (TypeScript)

Primary business logic for User Management and Core Data (CRUD).

Microservice BE

Golang

High-performance, isolated service (e.g., heavy processing, reporting, or notifications).

Database

SQL (PostgreSQL) + Prisma ORM

Primary source of truth for structured data (Users, Roles, Core Entities).

Database

NoSQL (MongoDB/Redis)

Flexible storage for user logs, feature flags, or caching.

4. Authentication Strategy (Hono Integration)

The Hono proxy will manage the entire social sign-in flow and act as the single point of entry for authorization checks.

Frontend (Next.js): Initiates the social sign-in flow (e.g., redirect to Google/GitHub) handled by the Hono proxy.

Hono Proxy: Completes the OAuth flow with the external provider. Upon successful authentication, it mints a JWT/session token containing the user's userId and role.

Hono Proxy: For all subsequent requests to the NestJS/Golang backend, the proxy validates the token and attaches the user context to the request header.

Backend (NestJS/Golang): The backend services trust the Hono proxy. They read the user context (userId, role) directly from the request headers inserted by the proxy and use the role to enforce business-level authorization checks.

5. Development Environment & Deployment Strategy

B. Hono API Gateway Best Practices

The Hono proxy serves as the API Gateway and must implement security and observability measures:

CORS Enforcement: Strictly configure CORS middleware to allow requests only from the Next.js frontend origin.

Request Context Injection: Inject the validated user data into standardized headers before forwarding to the backend:

X-User-ID: (Required for all services to identify the current user).

X-User-Role: (Required for all services to enforce RBAC).

X-Correlation-ID: (Generated here and passed down for distributed tracing/logging).

Rate Limiting: Implement basic rate limiting at the Hono layer to protect the backend services from abuse.

Input Validation (Mandatory): All endpoints must use Zod on the server side to rigorously validate and sanitize incoming data from the request body. Prisma's Client will handle type validation for all outgoing database queries, ensuring the data written to or read from PostgreSQL matches the defined schema.

Clear Error Handling: API responses must use standardized HTTP status codes (e.g., 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found) with simple, machine-readable error messages in the body.

C. Application Performance & Timeouts (5 Second SLA)

API Response Timeouts: All API calls across the system must enforce a maximum response time of 5 seconds. If this SLA is breached, the request must abort and return an appropriate error (e.g., 504 Gateway Timeout).

Implementation Strategy: Use Promise.race in both the Next.js frontend (with AbortController) and the NestJS/Golang backend for all outgoing network/database calls to enforce the 5-second limit.

D. Frontend Loading & UX Strategies

To ensure a highly responsive user experience, the following patterns are mandatory for the Next.js frontend:

Skeleton Screens: Use skeleton components to indicate loading state for lists, tables, and cards.

Optimistic UI Updates: Update the UI immediately upon a successful user action (like creating an entity) and handle rollback if the API call fails.

Error Boundaries: Implement React Error Boundaries to catch and display errors gracefully without crashing the entire application.

Preloading/Prefetching: Leverage Next.js's native prefetching (next/link) and smart data preloading for likely navigation targets.

E. Caching and Parallelism

Next.js Caching: Utilize SSG (getStaticProps) with ISR (revalidate) for public, non-user-specific pages. Use client-side data fetching libraries (SWR or React Query) for authenticated content, leveraging built-in caching and background revalidation.

Parallel I/O: Use Promise.all in all backend services (NestJS/Golang) and Next.js server components/API routes to parallelize independent I/O operations (DB queries, external API calls) to reduce response latency.

Multithreading: For any CPU-intensive tasks, use Node.js Worker Threads in Next.js/NestJS to avoid blocking the main event loop.

F. Type Safety & Data Modeling (Prisma Focus)

To ensure maximum type safety and align with the TypeScript-first approach, we are standardizing on Prisma ORM for all PostgreSQL interactions. Prisma follows a schema-first approach, generating a client that is fully aware of your database structure. This provides superior compile-time checks over traditional ORMs.

Example of Type-Safe Development:

Prisma's client is aware of the exact fields returned by a query. If we query a CoreEntity and only select the id and title, the resulting object will be automatically typed to contain only those two fields.

// Assume the prisma client is injected and available as 'prisma'
const userRole = 'Standard User'; 
const userId = 'user-123';

const entities = await prisma.coreEntity.findMany({
  where: { 
    // RBAC enforced: Standard users only see their own records
    ownerId: userRole === 'Standard User' ? userId : undefined,
  },
  select: {
    id: true,
    title: true,
  }
});

/* TypeScript automatically recognizes the structure of 'entities' as:
Array<{ id: string, title: string }>

Attempting to access a non-selected field, like 'entities[0].createdAt', 
will result in a compile-time error, ensuring data shape consistency.
*/

G. Database Migration Strategy (Prisma)

Migrations are a critical part of the Continuous Deployment (CD) pipeline. We must ensure the database schema is updated before the application starts using the new schema to prevent runtime errors.

Local Development (prisma migrate dev): Use this command locally to create new migration files, execute them against your development database, and update the generated Prisma Client. These migration files (in prisma/migrations) must be committed to Git.

npx prisma migrate dev --name your_migration_name


CI/CD Deployment (prisma migrate deploy): This command is mandatory in the pre-launch hook of the deployment pipeline. It runs all pending, non-applied migration files against the target environment (Staging/Production). This ensures the application version is compatible with the database schema.

# This must run before the new application container starts
npx prisma migrate deploy


H. Dockerization Strategy

To ensure consistency between development and production environments, all services will be containerized using Docker.

1.  **Local Development (Docker Compose)**:
    *   A `docker-compose.yml` file at the root of the monorepo will define the services for the entire stack (Next.js, NestJS, Hono, PostgreSQL, Redis).
    *   Developers will be able to spin up the entire development environment with a single command: `docker-compose up`.
    *   This approach isolates dependencies and eliminates the "it works on my machine" problem.
    *   Volumes will be used to mount local source code into the containers, enabling hot-reloading for a seamless development experience.

2.  **Production Deployment (Dockerfiles)**:
    *   Each application in the `apps/` directory will have its own multi-stage `Dockerfile`.
    *   These Dockerfiles will be optimized for production, creating small, secure, and efficient images.
    *   The CI/CD pipeline will use these Dockerfiles to build and push images to a container registry (e.g., Docker Hub, Google Container Registry).
    *   The deployment process will then pull these images to run in the production environment.


I. Circuit Breaker for 3rd Party Dependencies

To enhance resilience and prevent cascading failures caused by slow or failing third-party services, all external API calls (e.g., to payment gateways, email services, or other external APIs) must be wrapped in a circuit breaker.

-   **Pattern**: The circuit breaker will monitor calls to an external service. If the failure rate exceeds a configured threshold, the circuit will "open," and all subsequent calls will fail immediately without attempting to contact the service. This allows the external service time to recover.
-   **Implementation**: Use a library like **opossum** in the NestJS and Hono services.
-   **Fallback**: Implement sensible fallback mechanisms. For example, if a third-party data enrichment service is down, the system should gracefully continue without the enriched data rather than failing the entire request.


J. App Notifications (OneSignal)

For the MVP, all push notifications and in-app messaging will be handled by **OneSignal**. This allows for rapid implementation of a robust notification system.

-   **Implementation**: The NestJS monolith will be responsible for triggering transactional notifications (e.g., when a new entity is created or a user is mentioned). The Golang microservice can be used for high-volume or scheduled notifications.
-   **Frontend Integration**: The Next.js frontend will integrate the OneSignal SDK to handle the display of in-app messages and prompt users for push notification permissions.
-   **Channels**: The initial focus will be on Web Push (for desktop/mobile browsers) and In-App Messaging.


6. Observability & Testing Strategy

A. Structured Logging (The "More is More" Principle)

Use structured logging (JSON format) across all components (Next.js, Hono, NestJS, Golang) to ensure centralized aggregation and easy analysis.

Key Fields (Mandatory):

@timestamp (UTC, ISO 8601 format).

level (INFO, WARN, ERROR).

service (e.g., nestjs-monolith, golang-processor).

correlationId: Must be the value passed via the X-Correlation-ID header to link all logs for a single user request across the entire system.

Contextual Data: Include userId, API path, and latency/duration for performance monitoring.

Centralization: Configure services to write logs to stdout/stderr so they can be easily collected by the chosen cloud platform's logging service.

D. Performance Monitoring & Slack Integration

To provide real-time visibility into application health and performance, we will integrate a monitoring platform to send automated alerts to a dedicated Slack channel (e.g., `#system-alerts`).

-   **Tooling**: For the MVP, a managed service like **Datadog** or **Better Stack** is recommended for its ease of setup and powerful features. These services can directly ingest the structured JSON logs from our applications.
-   **Alerting Rules**: We will configure alerts for critical events, such as:
    -   Breaches of the 5-second SLA.
    -   Sudden increases in error rates.
    -   Unusual traffic patterns.
    -   High CPU or memory usage in containers.
-   **Alternative (Self-Hosted)**: A powerful open-source alternative is the **Prometheus**, **Grafana**, and **Alertmanager** stack. This would involve exposing a `/metrics` endpoint on our backend services and configuring Alertmanager to push notifications to Slack.

B. CI/CD Pipeline (GitHub Actions)

A single pipeline (ci.yml) should run on every Pull Request to ensure code quality and integration confidence.

Service

Test Type

Tooling

Purpose

Next.js

Unit/Component

Jest/React Testing Library

UI component reliability and utility logic.

NestJS

Unit/Integration

Jest/Supertest

Test services, controllers, and database repository interactions.

Golang

Unit/Integration

Go testing package

Test core functions and internal service logic.

Polyglot

Contract

Pact

Verify the API contracts between all services.

API Gateway

Security/Reliability

Supertest/Custom

Mandatory: Verify Rate Limiting, Idempotency, and Input Schema Validation.

C. Testing Strategy Breakdown

To ensure reliability and security, all testing efforts must adhere to the following specific guidelines for each test type:

1. Unit Testing

Purpose: Validate individual functions, components, and modules in isolation.

What to Test: Utility functions (price calculations, date/time logic), React components (UI rendering, props, conditional rendering), and API route handlers (input validation, error handling, business logic).

Recommended Tools: Jest for JavaScript/TypeScript unit tests; React Testing Library for React components.

Best Practices: Mock external dependencies (database, authentication), keep tests fast and focused, and use descriptive test names.

2. Integration Testing

Purpose: Test how multiple modules or components work together, including interactions with the database and authentication layer.

What to Test: API endpoints with real or in-memory database (entity creation, updates), Authentication and authorization flows (role-based access, protected routes), and Database migrations.

Recommended Tools: Jest or Vitest for running tests; Supertest for HTTP API testing; Prisma Test Utilities for database interactions.

Best Practices: Use a separate test database or in-memory database, reset database state between tests for isolation, and test both success and failure scenarios (e.g., unauthorized access, invalid data).

3. End-to-End (E2E) Testing

Purpose: Simulate real user interactions across the entire application stack, from frontend to backend and database.

What to Test: Key User Flows (login, entity creation/management, dashboard views), Role-based access verification (e.g., Admin can delete others' entities, Standard user cannot), and UI responsiveness.

Recommended Tools: Cypress or Playwright for E2E browser automation.

Best Practices: Run tests against a deployed staging environment, use realistic test accounts and data, and automate E2E tests in the CI/CD pipeline.

D. Deployment Pipeline (CD)

Build Stage: The CI job must build and tag Docker images for all backend services (NestJS, Golang, Hono).

Deployment Stage: On merge to main, GitHub Actions automatically pushes the container images to the container registry and triggers a deployment to the chosen cloud service (e.g., Cloud Run).

7. Phased Implementation Plan

Phase 1: Authentication Backbone (1 Week Target)

Auth Setup: Deploy Hono better-auth-proxy and configure the Social Sign-In Providers (e.g., GitHub, Google) to generate and validate tokens containing userId and role. Implement required gateway best practices (CORS, Header Injection, Rate Limiting).

NestJS Integration: Create the initial NestJS monolith structure. Implement middleware/guard to read and extract the user context (userId, role, correlationId) from the proxy-inserted headers. Implement 5-second timeout logic for all outgoing calls.

Frontend Shell: Set up the basic Next.js project structure, layout, and protected routing. Implement 5-second timeout and Skeleton loading on initial data fetch.

Phase 2: Core Business Logic (1-2 Weeks Target)

SQL Schema: Define the PostgreSQL schema for Users, Roles, and Core Entities using Prisma Schema Language (PSL).

NestJS CRUD: Implement the CoreEntity service in NestJS, including the business logic to enforce RBAC (i.e., filtering results based on the user's role/ID extracted from the headers). Implement Zod validation on all POST/PUT/PATCH endpoints.

Frontend MVP: Build the Dashboard and the form/list views necessary to test the Standard User and Admin permissions. Ensure Optimistic UI updates are used for key user actions.

Phase 3: Microservice Integration & Performance (1 Week Target)

Golang Microservice: Develop the initial Golang microservice that receives requests from the NestJS monolith via HTTP/gRPC. Implement the Golang side of the Contract Test.

Caching Implementation: Integrate Redis into the NestJS monolith. Implement caching for user role lookups to minimize database hits on every request. Apply SSG/ISR policies to public Next.js pages.

Testing & CD: Implement all Unit, Integration, and Contract Tests in the CI pipeline. Finalize the CD process for automated deployment, including tests for Rate Limiting and Idempotency.

8. Key Deliverables for MVP Launch

Fully functional Social Sign-In using the Hono proxy.

Next.js dashboard that successfully displays role-appropriate data.

NestJS service endpoints that strictly enforce Admin vs. Standard User CRUD permissions.

Hard 5-Second Response SLA enforced across the entire application stack.

All critical user interactions use Skeleton Screens or Optimistic UI updates.

Input Validation using Zod is mandatory on all mutation endpoints.

CI/CD pipeline running Unit, Integration, Contract, Rate Limiting, and Idempotency tests automatically.

All services implementing structured JSON logging with a request correlationId.

9. UI/UX and Component Structure

The Next.js frontend MUST adhere to the detailed aesthetic and structural requirements defined in the ui-preferences.md document. This includes:

Aesthetic: Implementing a custom, minimal CRM-style layout using direct Tailwind CSS classes for styling (e.g., gradients, rounded cards, sticky headers, backdrop blur).

Responsiveness: Full mobile/tablet/desktop responsiveness, including a collapsible sidebar.

Theme: Supporting Light/Dark mode with localStorage persistence.

Structure: Following the specified component organization (shared/, dashboard/, lib/, etc.).

10. Monorepo Strategy & Key Development Features

To streamline development and enhance collaboration, this project will adopt a monorepo architecture. All services (Next.js, NestJS, Hono, Golang) will reside in a single repository.

A. Monorepo Tooling

We will use **pnpm workspaces** as the primary tool for managing the monorepo. This choice is favored for its efficiency in handling node_modules and linking local packages. **Turborepo** will be layered on top to provide intelligent build caching, task orchestration, and simplified pipeline management.

B. Directory Structure

The monorepo will be organized as follows:

-   `apps/`: Contains the individual applications (e.g., `nextjs-frontend`, `nestjs-monolith`, `hono-proxy`).
-   `packages/`: Contains shared code and configurations accessible across the monorepo.

C. Key Development Features & Benefits

-   **Shared Code & Configuration**:
    -   `packages/ui`: Shared React components for the Next.js frontend.
    -   `packages/eslint-config-custom`: A single, shareable ESLint configuration.
    -   `packages/tsconfig`: Centralized TypeScript configurations to ensure consistency.
    -   `packages/zod-schemas`: Shared Zod schemas for validation between the frontend and backend, ensuring data consistency.

-   **Atomic Commits & Cross-Project Changes**:
    -   A single pull request can contain changes to both the frontend and backend, making it easier to coordinate features and bug fixes.

-   **Simplified Dependency Management**:
    -   Dependencies are managed from a single root `package.json`, reducing conflicts and ensuring version consistency.

-   **Efficient CI/CD**:
    -   Turborepo's caching and dependency graph analysis will significantly speed up build and test times in the CI/CD pipeline. Only the affected apps and packages will be rebuilt and tested.
