# Agnostic Notification Service Strategy

## 1. Core Principle: Abstraction and Decoupling

To ensure long-term flexibility and avoid vendor lock-in, the application's notification system is designed as a swappable, agnostic service. The core principle is to decouple the application's business logic from the specific implementation details of the notification provider (e.g., OneSignal, AWS SES, Twilio, etc.).

This is achieved by creating a dedicated **Notification Service** that acts as an abstraction layer.

## 2. Architectural Approach

The Notification Service exposes a consistent internal API to the rest of the system (e.g., the NestJS monolith). This API is generic and based on the *intent* of the notification, not the provider's specific features.

### Internal API Endpoints (Example)

- `POST /v1/notify/transactional`: For sending immediate, single-recipient notifications (e.g., "Your logbook has been signed.").
- `POST /v1/notify/batch`: For sending notifications to a list of users.
- `POST /v1/notify/broadcast`: For sending system-wide announcements.

### The "Provider" Model

Internally, the Notification Service will use a "provider" pattern.

- **Provider Interface**: A common interface defines the methods a provider must implement (e.g., `sendTransactional`, `sendBatch`).
- **Provider Implementations**: Concrete classes that implement the interface for specific services.
    - `OneSignalProvider`: Implements the logic to send notifications via the OneSignal API.
    - `AWSSESProvider`: Implements the logic to send emails via AWS Simple Email Service.
    - `StubProvider`: A mock provider for local development and testing that logs notifications to the console instead of sending them.

A configuration file determines which provider is currently active. Switching from OneSignal to AWS SES would only require changing a single environment variable (`NOTIFICATION_PROVIDER=AWS_SES`) and ensuring the corresponding provider is implemented.

## 3. Implementation

- **Service Location**: This logic can be encapsulated within the **Golang microservice** or a new, dedicated microservice to keep it isolated and independently scalable.
- **Configuration**: The active provider and its API keys/credentials will be managed via environment variables.
- **Communication**: The NestJS monolith will communicate with the Notification Service via simple HTTP requests or gRPC for higher performance.

## 4. Benefits

- **Flexibility**: Swap notification providers at any time with minimal code changes to the core application.
- **Testability**: Easily mock the notification service or use a `StubProvider` in testing environments.
- **Resilience**: The isolated service can have its own retry logic and circuit breakers, preventing notification provider outages from impacting the main application.
- **Cost Management**: Easily switch to a more cost-effective provider as the application scales.
