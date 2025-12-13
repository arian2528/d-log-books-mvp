SaaS MVP Development Instructions (instructions.md)

1. Project Goal & MVP Definition

Goal: To launch a Minimum Viable Product (MVP) for a role-based SaaS application designed to

[Core Business Value](./core-business-value.md)

MVP Definition (Key Differentiator): The MVP must enable users to perform the core value proposition and demonstrate the robust, role-based access control system.

2. Core MVP Features

The scope is strictly limited to the following features to ensure rapid deployment:

A. User Management & Authentication

Social Sign-In: Users will authenticate exclusively through a supported Social Provider (e.g., GitHub, Google) managed by the Hono proxy. No email/password flow.

Role Assignment: Initial users are assigned one of the following roles: Admin,Pilot, Mechanic (A&P), Operator, Owner, or Inspector (IA).

User Profile: Simple view/edit of name email and role.

B. Core Data Functionality (The "Do" Feature)

Log Entry Creation: Users can create log entries in the relevant logbooks (Operator, Airframe, Engine, etc.).

Role-Based Access Control (RBAC) & Signing:

*   **Pilot**: Can create and sign their own flight logs in the Pilot Log. This action triggers the cascading update.
*   **Operator**: Can manage operational data in the Operator Log, such as crew assignments and dispatch records.
*   **Mechanic (A&P)**: Can create and sign off on maintenance entries in the Airframe, Engine, Avionics and Propeller logs.
*   **Inspector (IA)**: Can perform and sign off on annual inspections and major repairs, making those records immutable.

C. Basic Dashboard

A landing page that displays a summary of the aircraft's status, including total times and upcoming maintenance items.

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

SQL (PostgreSQL) + TypeORM

Primary source of truth for structured data (Users, Roles, Core Entities).

Database

NoSQL (MongoDB/Redis)

Session Store for Hono Auth Proxy; Caching layer for backend services.

4. Authentication Strategy (Hono Integration)

The project uses a **hybrid authentication model** where the Hono proxy serves as the dedicated authentication server. This approach combines the security of stateful session cookies for the browser with the scalability of stateless JWTs for backend services.

This architecture is designed to be flexible and swappable, minimizing impact on the core application if the authentication technology changes in the future.

For a complete technical breakdown, including diagrams and implementation steps, refer to the single source of truth: **[Hybrid Authentication Strategy](./authentication-strategy.md)**.

5. Development Environment & Deployment Strategy

B. Hono API Gateway Best Practices

The Hono proxy serves as the API Gateway and must implement security and observability measures:

CORS Enforcement: Strictly configure CORS middleware to allow requests only from the Next.js frontend origin.

Request Context Injection: After validating the user's session cookie, the proxy mints a short-lived JWT and injects it into the downstream request's `Authorization` header.

`Authorization: Bearer <token>`: (Required for all backend services to authenticate the request).

`X-Correlation-ID`: (Generated here and passed down for distributed tracing/logging).

Rate Limiting: Implement basic rate limiting at the Hono layer to protect the backend services from abuse.

Input Validation (Mandatory): All endpoints must use Zod on the server side to rigorously validate and sanitize incoming data from the request body. TypeORM entities and repositories will handle type validation for all outgoing database queries, ensuring the data written to or read from PostgreSQL matches the defined schema.

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

F. Type Safety & Data Modeling (TypeORM Focus)

The complete database schema, including all tables, columns, and relationships, is defined in the [Database Schema Instructions](./database-schema.md) document. This document serves as the single source of truth for data modeling. All TypeORM entities must be implemented according to this specification.


G. Database Migration Strategy (TypeORM)

Migrations are critical for evolving the database schema in a controlled and versioned way.

Local Development:
Generate a new migration file based on changes in your entities.
`npx typeorm migration:generate -n YourMigrationName`

Run migrations to update your local database schema.
`npx typeorm migration:run`

CI/CD Deployment:
In a production environment, you should only run migrations. Generating migrations should be part of the development process. The CI/CD pipeline will run the migrations against the target database before the new application version is deployed.
`npx typeorm migration:run`

H. Row-Level Security (RLS) for Multi-Tenancy

To ensure strict data isolation between users (tenants), we will leverage PostgreSQL's native Row-Level Security (RLS). This is a robust way to enforce that users can only access their own data.

1. Enable RLS on the Table:
For each table that contains user-specific data, we need to enable RLS.
`ALTER TABLE core_entity ENABLE ROW LEVEL SECURITY;`

2. Create a Security Policy:
A policy is a rule that PostgreSQL applies to every query on the table. We will create a policy that checks the user's ID against a `userId` column in the table.

`CREATE POLICY user_isolation_policy ON core_entity
FOR ALL
USING (user_id = current_setting('app.current_user_id'));`

3. Set the User Context in the Application:
For every incoming request, we must tell PostgreSQL who the current user is. This is done by setting a session variable (`app.current_user_id`) at the beginning of each request. In our NestJS application, this can be done in a middleware.

// RLS middleware in NestJS
@Injectable()
export class RlsMiddleware implements NestMiddleware {
  constructor(private readonly connection: Connection) {}

  async use(req: Request, res: Response, next: NextFunction) {
    const userId = req.user?.id; // Assuming user is on the request
    if (userId) {
      await this.connection.query(`SET app.current_user_id = '${userId}'`);
    }
    next();
  }
}

With this setup, any query to `core_entity` from your application (using TypeORM) will automatically be filtered by PostgreSQL, so a user can only see and manipulate their own rows. This is a powerful and secure way to implement multi-tenancy.

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


J. App Notifications (Agnostic Service)

The application uses a provider-agnostic notification strategy to ensure flexibility and avoid vendor lock-in. The detailed approach, which involves an abstracted notification microservice, is described in the [Agnostic Notification Service Strategy](./notification-strategy.md) document.

-   **Initial Provider**: For the MVP, **OneSignal** will be the default provider for its rapid setup.
-   **Implementation**: A dedicated microservice (e.g., in Golang) will abstract the notification logic. The NestJS monolith will communicate with this service to trigger notifications, remaining decoupled from the specific provider implementation.
-   **Frontend Integration**: The Next.js frontend will integrate the necessary SDK (e.g., OneSignal's) to display notifications. The architecture ensures this can be swapped out with minimal effort.


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

Recommended Tools: Jest or Vitest for running tests; Supertest for HTTP API testing.

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

Phase 1: Project Scaffolding & Authentication Backbone (1 Week Target)

**Goal:** To establish the complete monorepo structure and implement the core hybrid authentication flow. For the granular, command-by-command setup for this phase, please refer to the official [MVP Build Instructions](./build-instructions.md).

-   **Monorepo Setup**: Initialize the pnpm workspace with Turborepo.
-   **Application Scaffolding**: Create the initial Next.js, NestJS, and Hono applications.
-   **Authentication Implementation**: Implement the stateful cookie-based auth between the frontend and proxy, and the stateless JWT-based auth between the proxy and the backend.
-   **Dockerization**: Containerize all services for a consistent local development environment.

Phase 2: Initial Deployment for Stakeholder Review (1 Week Target)

**Goal:** To deploy the skeleton application with the authentication backbone to a live environment. This will provide an early opportunity for stakeholders to see the progress and interact with the login flow.

-   **Deployment Strategy**: Follow the steps outlined in the **[MVP Phase Deployment Strategy](./deployment-strategy.md)** document.
-   **Frontend Deployment**: Deploy the Next.js application to Vercel.
-   **Backend Deployment**: Deploy the Hono proxy and NestJS monolith as containerized services on Google Cloud Run.
-   **Database Setup**: Provision the initial PostgreSQL database using a managed service like Supabase.
-   **CI/CD Pipeline**: Configure a basic CI/CD pipeline in GitHub Actions to automatically build and deploy the applications on merge to the `main` branch.

Phase 3: Core Business Logic (1-2 Weeks Target)

**Goal:** To implement the core features of the application, including the database schema and the CRUD (Create, Read, Update, Delete) operations that enforce the business rules. For a detailed breakdown of the required UI components and API endpoints for this phase, refer to the [Core Business Logic Implementation Guide](./core-business-implementation.md).

-   **SQL Schema**: Define the PostgreSQL schema for all entities as specified in the [Database Schema Instructions](./database-schema.md).
-   **NestJS CRUD**: Implement the CoreEntity service in NestJS. This includes the business logic to enforce RBAC by validating the incoming JWT and using its contents (e.g., `userId`, `role`) to control data access. Implement Zod validation on all POST/PUT/PATCH endpoints.
-   **Frontend MVP**: Build the Dashboard and the form/list views necessary. Ensure Optimistic UI updates are used for key user actions.

Phase 4: Microservice Integration & Performance (1 Week Target)

Golang Microservice: Develop the initial Golang microservice that receives requests from the NestJS monolith via HTTP/gRPC. Implement the Golang side of the Contract Test.

Caching Implementation: Integrate Redis into the NestJS monolith. Implement caching for user role lookups to minimize database hits on every request. Apply SSG/ISR policies to public Next.js pages.

Testing & CD: Implement all Unit, Integration, and Contract Tests in the CI pipeline. Finalize the CD process for automated deployment, including tests for Rate Limiting and Idempotency.

Phase 5: Monetization & Billing (Post-MVP)

Billing Microservice: Develop and deploy the dedicated, event-driven Billing Microservice as detailed in the [Agnostic Monetization & Billing Service](./monetization-strategy.md) document. Integrate the initial payment provider (e.g., Stripe).

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

## 11. Data Management

### A. ETL Strategy

For processes involving bulk data import, export, or synchronization, the project will follow a standardized, agnostic ETL (Extract, Transform, Load) approach. This ensures that data integration is consistent, scalable, and observable.

For a detailed guide on the ETL framework, including principles and recommended technologies, please see the [Agnostic ETL Process Strategy](./etl-strategy.md) document.

## 12. Post-MVP Enhancements

The following features are planned for implementation after the initial MVP launch. For detailed deployment strategies, including CDN implementation, please see the [Production Deployment Strategy](./deployment-strategy.md) document.


### B. API Versioning Strategy

As the application evolves and new features are added, maintaining backward compatibility for client applications (especially mobile or third-party integrations) becomes critical. A robust API versioning strategy will be implemented post-MVP to manage changes gracefully.

-   **Strategy**: **URL-based versioning** will be the chosen method. This is a clear and explicit approach where the version is included in the URL path.
    -   Example: `https://api.yourapp.com/v1/aircrafts`
    -   Example: `https://api.yourapp.com/v2/aircrafts`

-   **Implementation in NestJS**:
    -   NestJS provides built-in support for API versioning. We can enable it in the `main.ts` file and apply version numbers to controllers or individual routes.
    ```typescript
    // in main.ts
    app.enableVersioning({
      type: VersioningType.URI,
      defaultVersion: '1',
    });

    // in a controller
    @Controller({
      version: '2',
      path: 'aircrafts',
    })
    ```
    -   This allows different versions of a controller to coexist within the same application, simplifying the management of breaking changes.

-   **Gateway Routing**:
    -   The Hono API Gateway can be configured to route requests based on the version prefix in the URL to different backend services if, in the future, a new version of the API is built as a separate microservice.

-   **Best Practices**:
    -   **No Breaking Changes in Existing Versions**: Once an API version (e.g., `v1`) is public, it should not receive breaking changes. Bug fixes and non-breaking additions are acceptable.
    -   **Deprecation Policy**: When a new API version is released (e.g., `v2`), the old version (`v1`) should be marked as deprecated. A clear timeline for its eventual retirement should be communicated to all clients.
    -   **Clear Documentation**: Each version of the API must have its own clear and comprehensive documentation.

### C. API Documentation (Swagger/OpenAPI)

To ensure the API is well-documented and easy for developers (both internal and external) to consume, we will use Swagger/OpenAPI to automatically generate interactive API documentation.

-   **Strategy**: The NestJS application will generate an OpenAPI specification file based on the controllers, DTOs, and decorators in the code. This specification will be served via a UI.
-   **Implementation with `@nestjs/swagger`**:
    -   The `@nestjs/swagger` package will be used to instrument the application.
    ```typescript
    // in main.ts
    import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';

    async function bootstrap() {
      const app = await NestFactory.create(AppModule);

      const config = new DocumentBuilder()
        .setTitle('Aviation Logbook API')
        .setDescription('The official API documentation for the aviation logbook application.')
        .setVersion('1.0')
        .addBearerAuth()
        .build();
      const document = SwaggerModule.createDocument(app, config);
      SwaggerModule.setup('api-docs', app, document);

      await app.listen(3000);
    }
    bootstrap();
    ```
-   **Benefits**:
    -   **Always Up-to-Date**: The documentation is generated from the code, so it's always in sync with the API's actual implementation.
    -   **Interactive**: Developers can make live API calls directly from the documentation UI, which is excellent for testing and exploration.
    -   **Client SDK Generation**: The OpenAPI spec can be used to automatically generate client SDKs for various languages (e.g., TypeScript, Java).

## 12. Monetization Strategy

The application will be monetized based on user roles and access to features. The specific terms of use, pricing tiers, and subscription details will be defined at a later stage.

For the detailed technical implementation of the billing system, please refer to the [Agnostic Monetization & Billing Service](./monetization-strategy.md) document.

### D. Native Mobile Application Strategy

For the detailed approach to mobile development, including the progression from a Progressive Web App (PWA) to native applications, please refer to the [Mobile Application Strategy](./mobile-strategy.md) document.
