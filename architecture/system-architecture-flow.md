# 📐 System Architecture & Distributed Request Flow

This document details the end-to-end data flow, request lifecycle, cookie security model, transactional outbox pattern, distributed correlation tracking, and cloud-native Saga orchestration mechanism implemented across the e-commerce microservices platform.

---

## 🏗️ System Architecture Flow Diagram

```text
                       E-COMMERCE MICROSERVICES ARCHITECTURE
                                         │
                   🌐 Angular 20 Single Page App (localhost:4200)
                     ├── provideAppInitializer (Session Restore on F5 Page Refresh)
                     └── HTTP Interceptors (withCredentials, XSRF & 401 Silent Refresh)
                                         │
              (HTTPS + HttpOnly Cookies: accessToken & refreshToken + X-XSRF-TOKEN)
                                         ▼
                   🚪 API Gateway (localhost:5000)
                     ├── CorrelationMiddleware (Generates / Extracts X-Correlation-Id)
                     ├── XsrfCookieMiddleware (Anti-CSRF Protection)
                     ├── JwtForwardingHandler (Extracts HttpOnly Cookie -> Bearer Token)
                     ├── Sets HttpOnly cookies (BFF: browser gets { user } only)
                     └── CorrelationIdForwardingHandler (Propagates X-Correlation-Id Downstream)
                                         │
                 ┌───────────────────────┴───────────────────────┐
                 ▼                                               ▼
     🔐 Auth Service (localhost:4999)               📦 Order Service (localhost:5002)
   ├── BCrypt & JWT Token Rotation                 ├── Clean Architecture (.NET 10)
   ├── Hashed refresh tokens + reuse detection     ├── SQL Database + EF Core Outbox Table
   └── API Key Scope Management Endpoint           └── Multi-Scheme Auth & Dynamic Scope Security
                                                                 │
                                                        (Outbox Publisher)
                                                                 │
                                                                 ▼
                                               📢 AWS SNS Topic (Order Events Fanout)
                                                                 │
                                                                 ▼
                                               📥 AWS SQS Queue (Buffering & Dead-Letter Queue / DLQ)
                                                                 │
                                                                 ▼
                                               ⚡ AWS Lambda (Event Trigger & Saga Dispatcher)
                                                                 │
                                                                 ▼
                                               ⚙️ AWS Step Functions (Saga State Machine Orchestrator)
                                                                 │
                                                                 ├──────────────► 🏷️ Inventory Service (localhost:5003)
                                                                 │                 (Multi-Scheme Auth, Redis-Backed Scope Check,
                                                                 │                  Stock Check & Concurrency Reservation / Release)
                                                                 ▼
                                               ☁️ AWS CloudWatch Logs (Serilog Correlation Tracking)
                                                 (End-to-end request trace queryable via X-Correlation-Id)
```

---

## 🔄 Step-by-Step Request Lifecycle & Component Interaction

### 1. Client Request & Angular 20 Interceptor Pipeline (`localhost:4200` ➔ Gateway `localhost:5000`)
- **App Startup & Session Restore (`provideAppInitializer`):** When the browser loads or refreshes (F5), Angular runs `provideAppInitializer()` to execute `authService.initAuthSession()`, restoring user authentication state before page rendering.
- **Cookie Security (`jwtAuthInterceptor`):** All HTTP calls to the API Gateway set `withCredentials: true`. Access and Refresh tokens are stored securely in **HttpOnly, SameSite cookies** to prevent XSS token theft.
- **Anti-XSRF Protection:** On mutating HTTP requests (`POST`, `PUT`, `DELETE`), the interceptor reads the `XSRF-TOKEN` cookie set by the gateway and attaches the `X-XSRF-TOKEN` request header.
- **Silent Refresh on Expired Access Tokens (`errorInterceptor`):** If a business API request fails with `401 Unauthorized`, `errorInterceptor` intercepts the error, calls `/api/v1/auth/refresh` sending the HttpOnly `refreshToken` cookie, receives rotated cookies, and transparently re-sends the failed original request using RxJS `switchMap`.

### 2. API Gateway Middleware Pipeline (`ApiGateway.API` on Port 5000)
- **Correlation Tracking (`CorrelationMiddleware`):** Intercepts every incoming HTTP request. Extracts `X-Correlation-Id` header or generates a new `GUID` if absent. Pushes `CorrelationId` into Serilog `LogContext`.
- **Anti-Forgery Cookie Injection (`XsrfCookieMiddleware`):** Appends `XSRF-TOKEN` cookie to response headers for Angular CSRF validation.
- **Outbound HTTP Delegating Handlers:**
  - `JwtForwardingHandler`: Reads `accessToken` from HttpOnly cookies or `Authorization` header, converting it into an outbound `Authorization: Bearer <token>` header for downstream microservices.
  - `CorrelationIdForwardingHandler`: Copies `X-Correlation-Id` from current `HttpContext` onto outbound `HttpClient` requests calling downstream services.

### 3. Edge Authentication & Dynamic Scope Security Model (`Auth Service` on Port 4999)
- **JWT Issuance & Refresh Token Rotation:** AuthService issues a short-lived JWT access token and a refresh token. Only the SHA-256 hash of the refresh token is stored. A reused (already rotated) token revokes all of the user's sessions. The **API Gateway** (BFF) receives the tokens, writes them into HttpOnly `accessToken` and `refreshToken` cookies, and returns only `{ user }` to the browser.
- **Machine-to-Machine (M2M) & Dynamic Scope Security (`X-Api-Key` & `RequireScope`):** Automated M2M calls (e.g., AWS Step Functions SAGA orchestration calls to `InventoryService` or `OrderService`) authenticate via the `X-Api-Key` HTTP header.
  - **Redis Cache Validation**: `ApiKeyAuthHandler` uses `CachedApiKeyValidationService` to check the API key's scopes in Redis under `ecommerce:apikey:{SHA-256 hash of the key}`. If the key is not in Redis (cache miss), it calls `AuthService` (`/api/ApiKey/validate`) over HTTP and saves the result in Redis for 60 minutes.
  - **Dynamic Scope & Admin Authorization**: Endpoints are decorated with `[Authorize(AuthenticationSchemes = $"{JwtBearerDefaults.AuthenticationScheme},ApiKey")]` and `[RequireScope("...")]`. Admin users with a JWT token (`Role = "Admin"`) are allowed immediately with **0 Redis calls**, while SAGA machines are checked against their Redis-cached scopes (such as `inventory:reserve`, `inventory:release`, `order:update-status`).


### 4. Business Logic & Transactional Outbox (`Order Service` on Port 5002)
- Built following **Clean Architecture** (`Domain`, `Application`, `Infrastructure`, `API`).
- The controller receives the forwarded `Authorization: Bearer <token>` and `X-Correlation-Id`.
- **Transactional Outbox Pattern:**
  - The Order entity and an `OutboxMessage` containing the `OrderCreatedIntegrationEvent` are written to SQL Server inside a **single atomic database transaction**.
  - A background `OutboxPublisher` worker polls unpublished outbox messages, serializes event data, and publishes them to AWS SNS.

### 5. AWS Serverless Event Pipeline & Saga Execution (`SNS` ➔ `SQS` ➔ `Lambda` ➔ `Step Functions`)
- **AWS SNS Topic:** Receives `OrderCreatedIntegrationEvent` and fans out messages to **AWS SQS Queues**.

- **AWS SQS Queue:** Buffers messages asynchronously, equipped with a Dead-Letter Queue (DLQ) for failed message isolation and retry resilience.
- **AWS Lambda:** Triggered by SQS batches, parses event JSON, and invokes an **AWS Step Functions** state machine.
- **AWS Step Functions (Saga Orchestrator):** Manages multi-service transaction state machine:
  - Invokes **Inventory Service** (Port 5003) to check and reserve item stock using **Optimistic Concurrency Control** (`Version` / `RowVersion`).
  - Executes **Compensating Transactions** (e.g., releasing inventory stock or updating order status to `Cancelled`) if stock reservation fails or times out.

### 6. Centralized Microservice Observability (`Serilog` ➔ `AWS CloudWatch`)
- The microservices (`ApiGateway`, `OrderService`, `InventoryService`) use **Serilog** structured logging configured with AWS CloudWatch sink (`/ecommerce/gateway`, etc.).
- Every log entry carries `Service` (which service wrote it) and `CorrelationId` (pushed into Serilog `LogContext` by `CorrelationMiddleware`). Business values such as `OrderId` and `CustomerId` are written as structured properties of the log message.
- **End-to-End Log Tracing:** Searching a single `CorrelationId` (e.g. `ef873186-028d-4e48-abb3-42a206c36b1d`) in CloudWatch Insights reveals the step-by-step trace across API Gateway, Order Service, the Lambda dispatcher and Inventory Service.
