# All-In-One Aviation Record System

This repository contains the Minimum Viable Product (MVP) for a modern, role-based SaaS application designed to streamline aviation record-keeping. The system provides an integrated, all-in-one solution for pilots, mechanics, operators, and owners to manage aircraft logbooks with integrity and efficiency.

## Core Concept

The application's core value is its **"Cascading Update"** architecture. A single flight log entry by a pilot automatically propagates time updates across all related components—the airframe, engine(s), and propellers. This ensures data consistency, reduces manual entry errors, and provides an always-accurate view of the aircraft's operational status.

## Key Architectural Features

-   **Monorepo**: The project is structured as a monorepo using **pnpm workspaces** and **Turborepo** for efficient code sharing, dependency management, and build orchestration.
-   **Microservices-Oriented**: The architecture consists of a Next.js frontend, a NestJS monolith for core business logic, and a swappable **Hono** auth proxy. This design allows for flexibility and independent service scaling.
-   **Provider-Agnostic Services**: Key services like Authentication and Notifications are designed as abstracted, swappable components to avoid vendor lock-in.
-   **Data Integrity**: The system uses **PostgreSQL's Row-Level Security (RLS)** to enforce strict data isolation and **Zod** for rigorous server-side validation.

## Tech Stack

| Component              | Technology                               | Role                                                    |
| ---------------------- | ---------------------------------------- | ------------------------------------------------------- |
| **Frontend**           | Next.js (TypeScript)                     | Presentation layer & role-specific UI                   |
| **Auth Proxy**         | Hono                                     | Secure authentication gateway & token validation        |
| **Backend (Monolith)** | NestJS (TypeScript)                      | Core business logic, user management, and CRUD          |
| **Database**           | PostgreSQL + TypeORM                     | Primary source of truth for all structured data         |
| **Development**        | Docker Compose                           | Consistent local development environment                |
| **Deployment**         | Docker, Vercel, Google Cloud Run         | Containerized deployment for production                 |

## Getting Started

The entire development environment is containerized for consistency and ease of setup.

1.  **Clone the repository:**
    ```bash
    git clone https://github.com/arian2528/d-log-books-mvp.git
    cd d-log-books-mvp
    ```

2.  **Set up environment variables:**
    ```bash
    cp .env.example .env
    # Populate .env with your credentials
    ```

3.  **Run the application:**
    ```bash
    docker-compose up --build
    ```
    This command will build the images and start all the services (frontend, backend, database, etc.).

## Project Documentation

This `README.md` provides a high-level overview. For detailed technical specifications, architectural decisions, and step-by-step build instructions, please refer to the documents located in the **[.github](./.github)** directory.
