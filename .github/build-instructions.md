# Step-by-Step MVP Build Instructions

This document provides a granular, step-by-step guide to build and set up the initial phase of the application. These instructions are designed to be executed sequentially by a developer or an AI agent.

---

## Phase 0: Project & Monorepo Initialization

**Goal:** Set up the foundational monorepo structure using pnpm and Turborepo.

### Step 0.1: Initialize Project
1.  **Action:** Create the root `package.json`.
    ```bash
    pnpm init
    ```
2.  **Action:** Install Turborepo.
    ```bash
    pnpm add turbo --save-dev
    ```
3.  **Action:** Create the `pnpm-workspace.yaml` file at the project root.
    **File Content:**
    ```yaml
    packages:
      - 'apps/*'
      - 'packages/*'
    ```
4.  **Action:** Create the `turbo.json` file at the project root.
    **File Content:**
    ```json
    {
      "$schema": "https://turbo.build/schema.json",
      "globalDependencies": ["**/.env.*local"],
      "pipeline": {
        "build": {
          "dependsOn": ["^build"],
          "outputs": [".next/**", "!.next/cache/**", "dist/**"]
        },
        "lint": {},
        "dev": {
          "cache": false,
          "persistent": true
        }
      }
    }
    ```
5.  **Action:** Create the necessary directories.
    ```bash
    mkdir -p apps packages
    ```

### Step 0.2: Create Shared Packages
1.  **Action:** Create a shared `tsconfig` package for consistent TypeScript settings.
    - Create `packages/tsconfig/package.json`
    - Create `packages/tsconfig/base.json`
2.  **Action:** Create a shared `eslint-config-custom` package.
    - Create `packages/eslint-config-custom/package.json`
    - Create `packages/eslint-config-custom/index.js`

---

## Phase 1: Scaffolding Applications

**Goal:** Create the initial Next.js, NestJS, and Hono applications within the monorepo.

### Step 1.1: Create Next.js Frontend
1.  **Action:** Create the Next.js app.
    ```bash
    pnpm create next-app@latest apps/web --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"
    ```
2.  **Action:** Add the shared `tsconfig` and `eslint-config-custom` to `apps/web/package.json`.

### Step 1.2: Create NestJS Backend
1.  **Action:** Create the NestJS app.
    ```bash
    npx nest new apps/api --package-manager=pnpm
    ```
2.  **Action:** Add the shared `tsconfig` and `eslint-config-custom` to `apps/api/package.json`.

### Step 1.3: Create Hono Auth Proxy
1.  **Action:** Create a new directory `apps/auth-proxy` and initialize a `package.json`.
    ```bash
    mkdir -p apps/auth-proxy && cd apps/auth-proxy && pnpm init
    ```
2.  **Action:** Install Hono and TypeScript dependencies.
    ```bash
    pnpm add hono && pnpm add -D typescript @types/node
    ```
3.  **Action:** Create a basic `src/index.ts` file.

---

## Phase 2: Dockerization for Local Development

**Goal:** Containerize all services for a consistent local development environment.

### Step 2.1: Create Dockerfiles
1.  **Action:** Create `apps/web/Dockerfile` for the Next.js app. It should be a multi-stage build.
2.  **Action:** Create `apps/api/Dockerfile` for the NestJS app. It should be a multi-stage build.
3.  **Action:** Create `apps/auth-proxy/Dockerfile` for the Hono app.

### Step 2.2: Create Docker Compose
1.  **Action:** Create a `docker-compose.yml` file at the project root.
2.  **Action:** Define the following services in `docker-compose.yml`:
    - `postgres`: Using the official `postgres:15` image.
    - `api`: Building from `apps/api/Dockerfile`.
    - `web`: Building from `apps/web/Dockerfile`.
    - `auth-proxy`: Building from `apps/auth-proxy/Dockerfile`.
3.  **Action:** Configure volumes to mount source code for hot-reloading.
4.  **Action:** Define environment variables for all services, sourcing from a `.env` file.

### Step 2.3: Create Environment File
1.  **Action:** Create a `.env.example` file at the root with all necessary variables (DB credentials, Google OAuth keys, JWT secret, etc.).

---

## Phase 3: Authentication Backbone Implementation

**Goal:** Implement the Google Sign-In flow in the Hono proxy and set up the NestJS API to consume it.

### Step 3.1: Hono Auth Proxy Setup
1.  **Action:** In `apps/auth-proxy`, install Hono's JWT and Passport middleware.
2.  **Action:** Implement the Google OAuth2 strategy using Passport.
3.  **Action:** Create the `/auth/google` and `/auth/google/callback` endpoints.
4.  **Action:** In the callback, find or create a user in the database (via the NestJS API), then mint a JWT containing `userId` and `role`.
5.  **Action:** Create a Hono middleware to verify the JWT on incoming requests and inject `X-User-ID` and `X-User-Role` headers before proxying the request to the NestJS API.
6.  **Test:** Write a unit test for the JWT validation middleware.

### Step 3.2: NestJS API Setup
1.  **Action:** In `apps/api`, install TypeORM, pg, and Zod.
2.  **Action:** Configure the `AppModule` to connect to the PostgreSQL container using TypeORM.
3.  **Action:** Create `User` and `Role` TypeORM entities.
4.  **Action:** Generate the initial database migration.
    ```bash
    pnpm --filter api exec typeorm migration:generate -n InitialSchema
    ```
5.  **Action:** Create a NestJS middleware or guard to extract `X-User-ID` and `X-User-Role` from request headers and attach them to the request object.
6.  **Action:** Create a `users` module with a `/users/me` endpoint that returns the current user's data.
7.  **Test:** Write a unit test for the `/users/me` endpoint, mocking the request headers.

---

## Phase 4: Frontend and Admin Dashboard

**Goal:** Create a login page and a protected dashboard that displays a welcome message for the admin.

### Step 4.1: Frontend Login Page
1.  **Action:** In `apps/web`, create a login page at `/login`.
2.  **Action:** Add a "Sign in with Google" button that links to the Hono proxy's auth endpoint (`/auth/google`).
3.  **Action:** Create a callback page (e.g., `/auth/callback`) that receives the JWT from the proxy and stores it securely (e.g., in an HttpOnly cookie).

### Step 4.2: Protected Admin Dashboard
1.  **Action:** Create a dashboard page at `/dashboard`.
2.  **Action:** Implement client-side logic to check for the JWT. If it's not present, redirect to `/login`.
3.  **Action:** On the dashboard page, make an authenticated API call to the NestJS `/users/me` endpoint.
4.  **Action:** Display a welcome message, e.g., "Welcome, [User Name]! Your role is: [User Role]".
5.  **Test:** Create an E2E test using Cypress or Playwright that simulates the entire login flow:
    - Starts at the login page.
    - Clicks the "Sign in" button.
    - Mocks the Google OAuth callback to the Hono proxy.
    - Verifies the user is redirected to the dashboard.
    - Verifies the dashboard displays the correct user name and role.

---

## Phase 5: Finalizing the Local Environment

**Goal:** Ensure the entire stack runs correctly with a single command and that migrations and tests are integrated.

### Step 5.1: Run the Stack
1.  **Action:** Run the local environment.
    ```bash
    docker-compose up --build
    ```
2.  **Verification:** All services should start without errors. The NestJS API should automatically run the initial migration.

### Step 5.2: Integrate Testing
1.  **Action:** Add `test` and `test:e2e` scripts to the root `package.json` that use Turborepo to run tests across all applications.
    ```json
    "scripts": {
      "test": "turbo run test",
      "test:e2e": "turbo run test:e2e"
    }
    ```
2.  **Verification:** Run `pnpm test` and `pnpm test:e2e` to ensure all unit and end-to-end tests pass successfully.
