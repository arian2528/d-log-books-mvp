# Agnostic Monetization & Billing Service

This document outlines the technical strategy for implementing a flexible, event-driven monetization service that is decoupled from any single third-party payment provider.

## 1. Core Principles

-   **Abstraction & Decoupling**: The primary goal is to ensure the core application's business logic is not tightly coupled to a specific payment gateway like Stripe, Braintree, or Paddle. This allows for flexibility to switch providers or support multiple providers in the future with minimal code changes.
-   **Event-Driven Architecture**: The billing process will be handled asynchronously through an event-driven model. This enhances resilience, as billing-related events can be queued and processed independently, preventing payment provider outages from directly impacting core application performance.
-   **Dedicated Microservice**: All monetization logic will be encapsulated in a dedicated, standalone microservice. This isolates the complex and sensitive billing logic from the main application monolith.

## 2. Architectural Approach

### A. The Billing Microservice

A new microservice (e.g., written in Golang or NestJS) will be created to serve as the **Billing Service**. This service will be the single source of truth for all subscription and payment-related information. It will expose a simple, internal API for the main application to interact with.

### B. The "Provider" Pattern

Internally, the Billing Service will use a "provider" pattern, similar to the Notification Service.

-   **Billing Provider Interface**: A common interface will define the essential methods a payment provider must implement (e.g., `createSubscription`, `cancelSubscription`, `handleWebhook`, `getInvoice`).
-   **Concrete Provider Implementations**:
    -   `StripeProvider`: Implements the billing logic using the Stripe API and SDK.
    -   `PaddleProvider`: Implements the logic for the Paddle billing platform.
    -   `StubProvider`: A mock provider for local development and testing that logs billing actions to the console.

A configuration file (managed by environment variables) will determine the active provider. Switching from Stripe to Paddle would only require changing `PAYMENT_PROVIDER=PADDLE` and ensuring the `PaddleProvider` is implemented.

### C. Event-Driven Flow

The system will communicate via asynchronous events, likely using a message broker like RabbitMQ or a simple Redis queue.

1.  **Event Trigger**: The main NestJS application emits an event when a user performs a billing-related action (e.g., `user.subscribed`, `user.changed.plan`). The event payload contains necessary context, like `userId` and `planId`.
2.  **Event Consumption**: The Billing Service listens for these events.
3.  **Provider Interaction**: Upon receiving an event, the Billing Service calls the appropriate method on the currently active payment provider (e.g., `stripeProvider.createSubscription(userId, planId)`).
4.  **Webhook Handling**: The payment provider (e.g., Stripe) will send webhooks to a dedicated endpoint on the Billing Service to notify it of events that happen externally (e.g., `invoice.payment_succeeded`, `customer.subscription.deleted`). The Billing Service processes these webhooks and updates its own database.
5.  **Notification Integration**: After processing a critical billing event (like a successful payment or a failed charge), the Billing Service will emit another event (e.g., `billing.payment.failed`). The **Notification Service** will listen for these events and send the appropriate email or push notification to the user.

## 3. Benefits of this Approach

-   **Flexibility**: Avoids vendor lock-in with payment gateways.
-   **Resilience**: The main application remains responsive even if the payment provider's API is slow or down.
-   **Scalability**: The Billing Service can be scaled independently based on its specific load.
-   **Maintainability**: Isolates complex and critical billing logic into a single, focused service, making it easier to manage and secure.
