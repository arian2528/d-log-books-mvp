# Production Deployment Strategy

This document outlines the recommended deployment strategies for both the initial MVP launch and the post-MVP scaling phase of the application.

---

## 1. Guiding Principles

-   **Containerization**: All backend services are deployed as Docker containers, ensuring consistency between development and production environments.
-   **Infrastructure as Code (IaC)**: Whenever possible, infrastructure should be defined as code to ensure repeatability and version control.
-   **CI/CD Automation**: Deployments should be fully automated via a CI/CD pipeline (e.g., GitHub Actions).

---

## 2. MVP Phase Deployment Strategy

For the MVP, the primary goal is **speed of deployment, ease of management, and low operational overhead**. We recommend a combination of platforms that are optimized for specific parts of our stack and offer generous free/low-cost tiers.

### Recommended Providers:

-   **Frontend (Next.js)**: **Vercel**
-   **Backend Services (NestJS, Hono, Golang)**: **Google Cloud Run**
-   **Database (PostgreSQL)**: **Supabase** or **Render Postgres**

### Rationale & Implementation:

#### A. Frontend on Vercel

-   **Why**: Vercel is the creator of Next.js and offers a seamless, zero-configuration deployment experience. It provides automatic builds, previews for every Git push, a global edge network (CDN) out of the box, and scales automatically.
-   **Implementation**:
    1.  Connect the GitHub repository to a new Vercel project.
    2.  Vercel will automatically detect the Next.js application in the `apps/web` directory.
    3.  Configure the root directory in Vercel's project settings to point to `apps/web`.
    4.  Set the necessary environment variables (e.g., `NEXT_PUBLIC_API_URL`) in the Vercel dashboard.

#### B. Backend Services on Google Cloud Run

-   **Why**: Cloud Run is a fully managed, serverless platform that runs stateless containers. It automatically scales up or down, including to zero, meaning you only pay when your code is running. This is ideal for an MVP with variable traffic.
-   **Implementation**:
    1.  The CI/CD pipeline (GitHub Actions) will be configured to:
        -   Build Docker images for the NestJS, Hono, and Golang applications.
        -   Push these images to **Google Artifact Registry**.
        -   Trigger a new deployment on Google Cloud Run, pointing to the newly pushed image version.
    2.  Each service will have its own Cloud Run service, allowing them to be scaled and managed independently.
    3.  Environment variables (database URLs, JWT secrets, etc.) will be stored securely using Google Secret Manager and exposed to the Cloud Run services.

#### C. Database on Supabase

-   **Why**: Supabase provides a dedicated, managed PostgreSQL database with a user-friendly interface. It handles backups, security (SSL), and allows for easy scaling. For an MVP, this removes the burden of database administration. While Supabase offers its own auth and API services, we will only use its core database functionality.
-   **Implementation**:
    1.  Create a new project in Supabase to provision a PostgreSQL database.
    2.  Obtain the database connection string.
    3.  Add the connection string as a secret in Google Secret Manager for the backend services and as a secure environment variable for local development.
    4.  The NestJS API, when deployed, will run its TypeORM migrations against the Supabase database to set up the initial schema.

#### D. Custom Domains

-   **Frontend (Vercel)**: Vercel makes adding a custom domain straightforward. In the project settings, you can add your domain (e.g., `yourapp.com`) and Vercel will provide the DNS records (A or CNAME) to configure with your domain registrar.
-   **Backend (Google Cloud Run)**: To use a custom domain for your APIs (e.g., `api.yourapp.com`), you can map your Cloud Run services to a domain. This is typically done by setting up a Google Cloud Load Balancer and linking it to your services, then pointing your domain's DNS records to the load balancer's IP address.

---

## 3. Post-MVP Phase Deployment Strategy

As the application matures and traffic grows, the focus will shift towards **more control, advanced networking, and fine-grained scalability**. For this phase, we recommend migrating to a more robust, configurable infrastructure on a major cloud provider like **AWS** or **GCP**.

### Recommended Architecture: Kubernetes

-   **Platform**: **Amazon EKS (Elastic Kubernetes Service)** or **Google GKE (Google Kubernetes Engine)**.
-   **Why**: Kubernetes is the industry standard for container orchestration. It provides powerful capabilities for service discovery, load balancing, self-healing, and automated rollouts/rollbacks. This gives you maximum control over your application's architecture.

### Rationale & Implementation:

#### A. Container Orchestration with Kubernetes (EKS/GKE)

-   **Why**: Instead of running services on a serverless platform, you manage them within your own Kubernetes cluster. This is more complex but offers greater flexibility for managing stateful services, complex networking rules, and custom scaling policies.
-   **Implementation**:
    1.  **Infrastructure as Code (IaC)**: Use **Terraform** or **Pulumi** to define and provision the entire cloud infrastructure, including the VPC, subnets, Kubernetes cluster, and managed database.
    2.  **CI/CD Pipeline**: The GitHub Actions pipeline will be updated to:
        -   Build and push Docker images to a container registry (Amazon ECR or Google Artifact Registry).
        -   Use `kubectl` or a GitOps tool like **Argo CD** to apply Kubernetes manifest files (Deployments, Services, Ingress) to the cluster, triggering a rolling update of the applications.
    3.  **Ingress**: An Ingress Controller (like NGINX or Traefik) will be deployed to the cluster to manage external traffic routing to the various backend services.

#### B. Managed Database (RDS/Cloud SQL)

-   **Why**: For a production-grade database, using a fully managed service like **Amazon RDS for PostgreSQL** or **Google Cloud SQL** is recommended. These services offer high availability, automated backups, point-in-time recovery, read replicas, and robust security features.
-   **Implementation**: The database will be provisioned using the chosen IaC tool (Terraform/Pulumi) and its connection details will be securely injected into the Kubernetes cluster using a secret management system.

#### C. CDN for Static Asset Delivery

To improve global performance and reduce load on the Next.js server, a Content Delivery Network (CDN) like **AWS CloudFront** or **Cloudflare** will be implemented.

-   **Strategy**: The CDN will be configured to cache static assets generated by Next.js (`_next/static` directory), including JavaScript bundles, CSS files, and images.
-   **Implementation Steps**:
    1.  Configure the Next.js application to define the asset prefix to point to the CDN URL. This is done in `next.config.js`.
    2.  Set up a CDN distribution (e.g., in AWS CloudFront).
    3.  Configure the origin of the CDN to point to the Next.js application's domain.
    4.  Implement a cache-invalidation strategy in the CI/CD pipeline, so that new deployments automatically clear the CDN cache to serve the latest assets.
-   **Benefits**:
    -   **Reduced Latency**: Users will download assets from edge locations closer to them, resulting in faster page loads.
    -   **Improved Scalability**: Offloads traffic from the application server, allowing it to focus on handling dynamic requests.
    -   **Enhanced Reliability**: Provides an additional layer of redundancy.

This post-MVP approach requires more DevOps expertise but provides the foundation for a highly scalable, resilient, and secure application capable of handling enterprise-level workloads.
