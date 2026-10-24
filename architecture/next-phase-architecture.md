# 🚀 Next-Phase Architectural Roadmap

An overview of planned enhancements for the e-commerce microservice platform.

---

## 🗺️ Evolution Roadmap

```text
[1] DDD
[2] Polly Resilience
[3] CQRS with MediatR
[4] Distributed Idempotency
[5] Payment Service & SAGA Compensations
[6] API Gateway Rate Limiting
[7] Docker Compose Orchestration
```

---

## 📋 Planned Enhancements

* **1. DDD:** OrderService already uses `Entity` and `AggregateRoot` base classes. Next: extend the domain model further (e.g. Value Objects).
* **2. Fault Tolerance (Polly v8):** Implement automatic retries with exponential backoff and circuit breakers to prevent cascading HTTP network failures.
* **3. CQRS with MediatR:** Separate read queries from write commands while running centralized validation and logging pipeline behaviors.
* **4. Distributed Idempotency (Redis):** Use `X-Idempotency-Key` headers and Redis distributed locks to prevent duplicate orders and double billing.
* **5. Payment Service & SAGA Compensations:** Add a dedicated Stripe Payment Service with AWS Step Functions compensating refunds on payment failures or 5-minute timeouts.
* **6. API Gateway Rate Limiting:** Enforce IP and user-based rate limit policies at the gateway to protect against abuse and traffic spikes.
* **7. Multi-Container Orchestration (Docker Compose):** Configure a single `docker-compose.yml` to spin up all microservices, SQL Server, Redis, and local AWS emulators in one command.
