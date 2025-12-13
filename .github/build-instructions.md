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

**Goal:** Implement the hybrid authentication flow. The Hono proxy will manage `HttpOnly` session cookies with the Next.js frontend and communicate with the NestJS backend via short-lived JWTs.

### Step 3.1: Hono Auth Proxy Setup (Stateful Leg)
1.  **Action:** In `apps/auth-proxy`, install dependencies for session management and JWT creation.
    ```bash
    pnpm --filter auth-proxy add better-auth-hono ioredis jsonwebtoken
    pnpm --filter auth-proxy add -D @types/jsonwebtoken
    ```
2.  **Action:** Configure `better-auth` with the Redis adapter to handle Google OAuth and session storage.
3.  **Action:** Implement the `/auth/google` and `/auth/google/callback` endpoints.
4.  **Action:** In the callback, after `better-auth` validates the user, it will automatically create a session in Redis and set a secure, `HttpOnly` cookie on the user's browser before redirecting them back to the Next.js app.
5.  **Action:** Create a "translation" middleware in Hono. This middleware will:
    a.  Verify the incoming session cookie using `better-auth`'s API.
    b.  If the session is valid, mint a new, short-lived JWT (`~5-15 minutes`) containing the `userId` and `role`.
    c.  Sign the JWT with a secret shared only with the NestJS API.
    d.  Attach the JWT to the downstream request as an `Authorization: Bearer <token>` header before proxying the request to the NestJS API.
6.  **Test:** Write a unit test for the "translation" middleware to ensure it correctly validates a session and attaches a valid JWT.

### Step 3.2: NestJS API Setup (Stateless Leg)
1.  **Action:** In `apps/api`, install NestJS JWT, Passport, Swagger, and logging packages.
    ```bash
    pnpm --filter api add @nestjs/passport @nestjs/jwt passport passport-jwt @nestjs/swagger pino-http pino-pretty
    pnpm --filter api add -D @types/passport-jwt
    ```
2.  **Action:** In `apps/api/src/main.ts`, enable API versioning and set up Swagger documentation.
    **File Content (`main.ts`):**
    ```typescript
    import { NestFactory } from '@nestjs/core';
    import { AppModule } from './app.module';
    import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';
    import { VersioningType } from '@nestjs/common';
    import { Logger } from 'nestjs-pino';

    async function bootstrap() {
      const app = await NestFactory.create(AppModule, { bufferLogs: true });
      
      // Use Pino for structured logging
      app.useLogger(app.get(Logger));

      // Enable URI-based API versioning
      app.enableVersioning({
        type: VersioningType.URI,
        defaultVersion: '1',
      });

      // Configure Swagger/OpenAPI
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
3.  **Action:** Configure the `AuthModule` to use the `JwtStrategy` from `passport-jwt`. The strategy should be configured to validate the `Bearer` token using the same secret as the Hono proxy.
4.  **Action:** In `main.ts`, configure `pino-http` as the logger to enforce structured JSON logging.
5.  **Action:** Create a `JwtAuthGuard` that can be used to protect routes. This guard will automatically validate the incoming JWT and attach the payload (containing `userId` and `role`) to the request object.
6.  **Action:** Create a `users` module with a `/users/me` endpoint protected by the `JwtAuthGuard`. Apply a version to its controller.
    **Example Controller:**
    ```typescript
    @Controller({
      version: '1',
      path: 'users',
    })
    export class UsersController {
      // ...
      @Get('me')
      @UseGuards(JwtAuthGuard)
      getProfile(@Request() req) {
        return req.user;
      }
    }
    ```
7.  **Test:** Write a **unit test** for the `/users/me` endpoint, mocking the `Authorization` header with a valid JWT.
8.  **Test:** Write an **integration test** for the `JwtAuthGuard` to ensure it properly protects routes and validates tokens.

---

## Phase 4: Frontend and Admin Dashboard

**Goal:** Create a login page and a protected dashboard that relies on the browser's automatic handling of `HttpOnly` cookies.

### Step 4.1: Frontend Login Page
1.  **Action:** In `apps/web`, create a login page at `/login`.
2.  **Action:** Add a "Sign in with Google" button. This button should be a simple `<a>` tag linking directly to the Hono proxy's auth endpoint (e.g., `http://localhost:8787/auth/google`).
3.  **Action:** The Next.js app does not need a dedicated callback page. After successful login, the Hono proxy will redirect the user directly to a page of your choice, like `/dashboard`, with the session cookie already set.

### Step 4.2: Protected Admin Dashboard
1.  **Action:** Create a dashboard page at `/dashboard`.
2.  **Action:** The frontend is now "auth-dumb." It does not need to check for a JWT. To fetch data, it simply makes an API call to the Next.js backend (which proxies to Hono, then to NestJS). The browser automatically includes the `HttpOnly` cookie with the request.
3.  **Action:** On the dashboard page, make an API call to `/api/users/me` (a proxied route).
4.  **Action:** If the API call is successful, display the user's data: "Welcome, [User Name]! Your role is: [User Role]". If the call fails with a 401 Unauthorized, it means the session is invalid, and the user should be redirected to `/login`.
5.  **Test:** Create an E2E test using Cypress or Playwright that simulates the entire login flow:
    - Starts at the login page.
    - Clicks the "Sign in" button.
    - Mocks the Google OAuth flow.
    - Verifies the user is redirected to the dashboard.
    - Verifies the dashboard successfully fetches and displays the correct user name and role.

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
