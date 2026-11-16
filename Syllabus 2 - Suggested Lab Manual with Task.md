### Unit 1: Microservices & Design Architecture (Practicals 1 – 5)

*(Focus: Distributed system basics, Docker infrastructure, Database-per-Service, and synchronous REST/gRPC communication)*

#### Practical 1: Setting Up Microservices Infrastructure using Docker Compose

* **Aim:** To configure isolated PostgreSQL databases, Redis, and RabbitMQ instances using Docker Compose and scaffold a multi-service ASP.NET Core solution.
* **Description:** Students create a root solution (`ECommercePlatform.sln`) with separate Web API projects (`Catalog.API` and `Order.API`) and write a `docker-compose.yml` file that spins up two isolated PostgreSQL containers (`catalog-db` on port `5433`, `order-db` on port `5434`), Redis (`6379`), and RabbitMQ (`5672` / `15672`).
* **Output Flow:**
1. Student runs `docker-compose up -d` in the terminal.
2. `docker ps` displays all 4 infrastructure containers running with healthy statuses.
3. Students verify connectivity to both PostgreSQL instances using pgAdmin/DBeaver and open the RabbitMQ Management UI at `http://localhost:15672`.



#### Practical 2: Implementing `Catalog.API` with Database-per-Service Pattern

* **Aim:** To develop the Product Catalog microservice with its own isolated PostgreSQL database (`CatalogDb`) using Entity Framework Core and expose product CRUD endpoints.
* **Description:** Students implement the `Product` domain entity (`Id`, `Name`, `Price`, `StockQuantity`, `UpdatedAt`), configure `CatalogDbContext` connected exclusively to `catalog-db`, run EF Core migrations, and build REST endpoints (`GET`, `POST`, `PUT`, `DELETE /api/products`).
* **Output Flow:**
1. Student runs `dotnet ef database update` to generate the `Products` table inside `CatalogDb`.
2. Student sends a `POST /api/products` request via Postman/Swagger to create `"Wireless Mouse"` (`Price: 1500`, `StockQuantity: 25`).
3. API returns `201 Created`, and `GET /api/products` fetches the newly inserted record from `CatalogDb`.



#### Practical 3: Implementing `Order.API` with Clean Architecture

* **Aim:** To build the Order microservice (`Order.API`) with its own dedicated database (`OrderDb`) without sharing database tables or foreign keys with `Catalog.API`.
* **Description:** Students structure `Order.API` into Domain, Application, and Infrastructure layers. They create `Order` and `OrderItem` entities storing `ProductId` as a plain `UUID` (no cross-database foreign key), apply migrations to `OrderDb`, and implement `POST /api/orders` and `GET /api/orders/{id}`.
* **Output Flow:**
1. Student runs migrations on `OrderDb` and inspects the schema to confirm `Orders` and `OrderItems` exist independently of `CatalogDb`.
2. Student calls `POST /api/orders` with a product ID and quantity.
3. `Order.API` saves the order with `Status = "Pending"` and returns `201 Created` with the generated `OrderId`.



#### Practical 4: Synchronous Service-to-Service Communication using Typed `HttpClient`

* **Aim:** To enable `Order.API` to fetch real-time product pricing and stock availability from `Catalog.API` over HTTP before creating an order.
* **Description:** Students register a typed `ICatalogServiceClient` in `Order.API` using `IHttpClientFactory`. When `POST /api/orders` is called, `Order.API` makes a synchronous `GET /api/products/{id}` call to `Catalog.API`, validates that `StockQuantity >= requestedQuantity`, calculates `TotalAmount` using the live price, and saves the order.
* **Output Flow:**
1. Student sends `POST /api/orders` requesting `Quantity: 2` for a product priced at `1500`.
2. `Order.API` queries `Catalog.API` over HTTP, verifies stock (`25 >= 2`), and saves `TotalAmount = 3000`.
3. Student tests requesting `Quantity: 100` (exceeding stock) and verifies `Order.API` rejects the request with `400 Bad Request ("Insufficient stock")`.



#### Practical 5: High-Performance Inter-Service Communication using gRPC

* **Aim:** To implement a binary gRPC endpoint in `Catalog.API` for fast stock verification and compare its payload size and latency against REST/JSON.
* **Description:** Students define a `catalog.proto` contract (`GetProductPriceAndStock`), generate C# server/client stubs using `Grpc.AspNetCore`, host the gRPC service inside `Catalog.API`, and add a gRPC client call option in `Order.API`.
* **Output Flow:**
1. Student triggers an order placement using the gRPC client path in `Order.API`.
2. `Catalog.API` logs an incoming `HTTP/2 application/grpc` request and returns the binary protobuf response.
3. Student compares Postman/Swagger REST response size (~350 bytes JSON) vs. gRPC binary payload (~28 bytes) in the lab journal.



---

### Unit 2: Caching, Rate Limiting, Logging and Security (Practicals 6 – 11)

*(Focus: Redis caching, Fixed Window vs. Token Bucket throttling, Serilog structured logging, and JWT Auth)*

#### Practical 6: Distributed Read-Through Caching & Cache Invalidation with Redis

* **Aim:** To reduce database load in `Catalog.API` by caching product responses in Redis and invalidating stale cache entries upon updates.
* **Description:** Students configure `AddStackExchangeRedisCache` in `Catalog.API`. Inside `GET /api/products/{id}`, the service checks Redis (`product:{id}`) before querying PostgreSQL and stores misses with a 5-minute expiration. Inside `PUT` and `DELETE`, `RemoveAsync("product:{id}")` is called.
* **Output Flow:**
1. First `GET /api/products/{id}` logs `[Cache MISS - Querying PostgreSQL]` and takes ~45ms.
2. Second `GET /api/products/{id}` logs `[Cache HIT - Returned from Redis]` and completes in less than 5ms.
3. Student calls `PUT /api/products/{id}` to change the price, then calls `GET` again—verifying a fresh cache miss occurs and the updated price is returned.



#### Practical 7: Implementing Fixed Window Rate Limiting on `Catalog.API`

* **Aim:** To protect public product browsing endpoints from scraping and traffic bursts using ASP.NET Core's Fixed Window rate limiter.
* **Description:** Students configure `AddFixedWindowLimiter` in `Catalog.API` partitioned by client IP address, permitting a maximum of `5 requests per 30 seconds` with `QueueLimit = 0`.
* **Output Flow:**
1. Student sends 5 rapid `GET /api/products` requests in Postman; all 5 return `200 OK`.
2. On the 6th request within the 30-second window, `Catalog.API` rejects the call with `429 Too Many Requests` and a `Retry-After` header.
3. After waiting 30 seconds, the window resets and requests succeed again.



#### Practical 8: Implementing Token Bucket Rate Limiting on `Order.API`

* **Aim:** To control order submission bursts using a Token Bucket algorithm and compare its traffic-smoothing behavior against Fixed Window limiting.
* **Description:** Students configure `AddTokenBucketLimiter` on `POST /api/orders` with `TokenLimit = 4`, `TokensPerPeriod = 1`, and `ReplenishmentPeriod = 10 seconds`, allowing short bursts of 4 orders while refilling capacity gradually.
* **Output Flow:**
1. Student fires 4 rapid `POST /api/orders` requests; all 4 consume bucket tokens and succeed (`201 Created`).
2. The 5th immediate request returns `429 Too Many Requests`.
3. Student waits 10 seconds (1 token replenished) and sends 1 request, verifying it succeeds immediately without waiting for a full window reset.



#### Practical 9: Structured Logging & Correlation ID Propagation with Serilog

* **Aim:** To replace plain-text console logs with structured JSON logs using Serilog and track requests across microservices using an `X-Correlation-ID` header.
* **Description:** Students install `Serilog.AspNetCore` in both `Catalog.API` and `Order.API`, write custom middleware that generates or reads `X-Correlation-ID`, enriches `LogContext`, and forwards the header via a `DelegatingHandler` on outgoing `HttpClient` calls.
* **Output Flow:**
1. Student sends `POST /api/orders` with header `X-Correlation-ID: lab-trace-101`.
2. `Order.API` outputs structured JSON logs containing `"CorrelationId": "lab-trace-101"`.
3. `Catalog.API` receives the forwarded header and logs `"CorrelationId": "lab-trace-101"`, allowing students to filter all logs for that single transaction across both services.



#### Practical 10: Building `Identity.API` for JWT Token Generation

* **Aim:** To implement a dedicated Authentication microservice (`Identity.API`) that validates user credentials and issues signed JSON Web Tokens (JWT).
* **Description:** Students create `Identity.API` with its own `IdentityDb`, implement user registration (`POST /api/auth/register`) with password hashing, and build `POST /api/auth/login` to generate HMAC-SHA256 signed JWTs embedding `sub` (UserId), `email`, and `role` (`Admin` or `Customer`) claims.
* **Output Flow:**
1. Student registers two users: `admin@lab.com` (`Role: Admin`) and `student@lab.com` (`Role: Customer`).
2. Student calls `POST /api/auth/login` and receives a valid `access_token`.
3. Student decodes the token on `jwt.io` (or via code) to verify the embedded `sub`, `role`, and `exp` claims.



#### Practical 11: Enforcing Role-Based Authorization Across Microservices

* **Aim:** To secure `Catalog.API` and `Order.API` endpoints using JWT Bearer validation and enforce role-based access control (`Admin` vs. `Customer`).
* **Description:** Students configure `AddAuthentication().AddJwtBearer()` in both services using the shared signing key and apply `[Authorize(Roles = "Admin")]` on `POST/PUT/DELETE /api/products` and `[Authorize(Roles = "Customer")]` on `POST /api/orders`.
* **Output Flow:**
1. Calling `POST /api/products` without a token returns `401 Unauthorized`.
2. Calling `POST /api/products` using the `Customer` JWT token returns `403 Forbidden`.
3. Calling `POST /api/products` using the `Admin` JWT token succeeds (`201 Created`), and `Customer` successfully places an order on `POST /api/orders`.



---

### Unit 3: API Gateway, Resilience and Distributed Systems (Practicals 12 – 17)

*(Focus: Ocelot API Gateway, Polly Retry & Circuit Breaker, Redis Idempotency, Saga Pattern, and OpenTelemetry)*

#### Practical 12: Configuring a Unified API Gateway using Ocelot

* **Aim:** To hide internal microservice ports behind a single entry point (`Gateway.API`) using Ocelot and centralize upstream routing and JWT authentication.
* **Description:** Students create `Gateway.API`, install the `Ocelot` package, configure `ocelot.json` to map `/gateway/catalog/{everything}` and `/gateway/orders/{everything}` to internal ports, and configure JWT validation at the Gateway level.
* **Output Flow:**
1. Student sends requests exclusively to `http://localhost:5000/gateway/catalog/products` instead of internal service ports.
2. Ocelot validates the Bearer token at the gateway edge and proxies the request to `Catalog.API`.
3. Unauthenticated requests are rejected directly by Ocelot (`401 Unauthorized`) without hitting downstream microservices.



#### Practical 13: Implementing Retry & Timeout Policies with Polly

* **Aim:** To make service-to-service HTTP calls resilient against temporary network glitches using Exponential Backoff Retries and Timeouts.
* **Description:** Students configure `AddStandardResilienceHandler()` (`Microsoft.Extensions.Http.Resilience` / Polly) on `ICatalogServiceClient` in `Order.API` with 3 retries (`200ms`, `400ms`, `800ms` exponential backoff) and a `2-second` attempt timeout.
* **Output Flow:**
1. Student adds a simulated transient failure (returning HTTP `503` on the first 2 attempts) in `Catalog.API`.
2. Student calls `POST /gateway/orders`.
3. Console logs show Polly automatically retrying attempt 1 and attempt 2, succeeding on attempt 3, and returning `201 Created` to the client without error.



#### Practical 14: Implementing Circuit Breaker & Graceful Degradation

* **Aim:** To prevent cascading failures during prolonged `Catalog.API` downtime by configuring a Polly Circuit Breaker policy.
* **Description:** Students configure a Circuit Breaker strategy that trips to the `Open` state for `20 seconds` if 50% of requests fail within a 15-second window, catching `BrokenCircuitException` in `Order.API` to return a clean fallback response.
* **Output Flow:**
1. Student stops the `Catalog.API` container/process and sends 5 order requests.
2. After the failure threshold is reached, logs print `Circuit Breaker transitioned to OPEN`.
3. Subsequent requests fail fast in less than 2ms with `503 Service Unavailable ("Catalog service is temporarily down, please retry later")` without waiting for network timeouts.



#### Practical 15: Preventing Duplicate Orders using Redis Idempotency Keys

* **Aim:** To ensure duplicate `POST /orders` retries do not create duplicate database records by implementing an Idempotency filter backed by Redis.
* **Description:** Students build an `IdempotencyFilter` in `Order.API` that inspects the `Idempotency-Key` request header, checks Redis (`idempotency:{userId}:{key}`), returns the cached HTTP response if already processed, or executes and stores the result for 24 hours.
* **Output Flow:**
1. Student sends `POST /gateway/orders` with `Idempotency-Key: txn-999`; `Order.API` creates Order `#101` (`201 Created`).
2. Student resends the exact same request with `Idempotency-Key: txn-999`.
3. `Order.API` bypasses database insertion, returns the exact same Order `#101` response from Redis, and `OrderDb` contains only 1 order row.



#### Practical 16: Implementing Distributed Transactions via Choreography Saga

* **Aim:** To maintain cross-database consistency between `Order.API` and `Catalog.API` using an event-driven Choreography Saga with compensating transactions.
* **Description:** Students replace synchronous stock deduction with a Saga flow: `Order.API` saves the order as `Pending` and publishes `OrderCreatedEvent`; `Catalog.API` deducts stock and publishes either `StockReservedEvent` or `StockReservationFailedEvent`; `Order.API` updates the order to `Confirmed` or runs a compensating update to `Cancelled`.
* **Output Flow:**
1. **Happy Path:** Student orders `2` units (in stock). Order starts as `Pending`, `CatalogDb` stock drops by `2`, and `OrderDb` status updates to `Confirmed`.
2. **Compensation Path:** Student orders `500` units (out of stock). Order starts as `Pending`, `Catalog.API` publishes `StockReservationFailedEvent`, and `Order.API` automatically rolls back the order status to `Cancelled`.



#### Practical 17: Distributed Tracing across Microservices with OpenTelemetry & Jaeger

* **Aim:** To trace a single distributed transaction across Ocelot, HTTP calls, PostgreSQL, Redis, and message queues using OpenTelemetry and Jaeger.
* **Description:** Students add Jaeger to `docker-compose.yml` (`port 16686`) and configure OpenTelemetry tracing in `Gateway.API`, `Order.API`, and `Catalog.API` with ASP.NET Core, `HttpClient`, EF Core, and OTLP exporters.
* **Output Flow:**
1. Student places an order through `Gateway.API`.
2. Student opens the Jaeger UI (`http://localhost:16686`), searches for the service trace, and inspects the unified `TraceId` waterfall showing exact millisecond durations across Gateway $\rightarrow$ Order.API $\rightarrow$ Redis $\rightarrow$ Catalog.API $\rightarrow$ PostgreSQL.



---

### Unit 4: Background Jobs, Queuing & Messaging (Practicals 18 – 21)

*(Focus: RabbitMQ Producer–Consumer architecture, Dead-Letter Queues, and Hangfire background scheduling)*

#### Practical 18: Producer–Consumer Messaging with RabbitMQ & MassTransit

* **Aim:** To implement decoupled asynchronous communication using RabbitMQ exchanges, durable queues, and MassTransit producers/consumers.
* **Description:** Students configure MassTransit with RabbitMQ transport across `Order.API` and `Catalog.API`, explicitly defining topic exchanges, queue bindings, and strongly typed message contracts (`OrderCreatedEvent`, `StockReservedEvent`).
* **Output Flow:**
1. Student starts both services and inspects the RabbitMQ Management UI (`Exchanges` and `Queues` tabs) to verify auto-provisioned durable queues.
2. Student stops `Catalog.API`, places 3 orders in `Order.API`, and observes `Ready: 3` messages queued safely in RabbitMQ.
3. Student restarts `Catalog.API` and observes all 3 queued messages immediately consumed and processed (`Ready: 0`).



#### Practical 19: Fault-Tolerant Consumers & Dead-Letter Queues (DLQ)

* **Aim:** To handle poison messages and transient database errors during queue consumption using retry policies and Dead-Letter Queues in RabbitMQ.
* **Description:** Students configure `UseMessageRetry(r => r.Intervals(1000, 3000, 5000))` on their MassTransit consumer and simulate an unhandled exception when a specific test product ID is processed.
* **Output Flow:**
1. Student publishes an event containing the simulated poison payload.
2. Console logs show MassTransit retrying the message 3 times at 1s, 3s, and 5s intervals.
3. After retries are exhausted, RabbitMQ routes the failed message to the `order-created_error` Dead-Letter Queue, keeping the main queue unblocked for valid orders.



#### Practical 20: Background Job Processing & Fire-and-Forget Jobs using Hangfire

* **Aim:** To offload non-blocking post-order processing (PDF/Text invoice generation) using Hangfire backed by PostgreSQL.
* **Description:** Students install `Hangfire.AspNetCore` and `Hangfire.PostgreSql` in `Order.API`, secure the `/hangfire` dashboard, and enqueue a Fire-and-Forget job (`BackgroundJob.Enqueue`) upon order confirmation to generate an invoice file and update `Orders.InvoiceUrl`.
* **Output Flow:**
1. When an order transitions to `Confirmed`, `Order.API` immediately returns without waiting for file generation.
2. Student opens `http://localhost:<port>/hangfire`, views the job under `Succeeded Jobs`, and verifies `InvoiceUrl` is populated in `OrderDb`.



#### Practical 21: Delayed & Recurring Scheduled Jobs in Hangfire

* **Aim:** To automate unpaid order expiration using Hangfire Delayed Jobs and daily sales aggregation using CRON Recurring Jobs.
* **Description:** Students schedule a Delayed Job (`BackgroundJob.Schedule(..., TimeSpan.FromMinutes(2))` for lab testing) when an order is created that cancels the order if it remains `Pending`, and register a `RecurringJob` (`Cron.Daily`) that computes daily revenue summaries.
* **Output Flow:**
1. Student creates an order while `Catalog.API` is offline so the order stays in `Pending`.
2. After 2 minutes, the Hangfire Delayed Job executes automatically and transitions the order status to `Cancelled`.
3. Student manually triggers the `daily-sales-summary` Recurring Job from the Hangfire Dashboard and verifies the aggregated revenue log output.



---

### Unit 5: Cloud & Realtime Communication (Practicals 22 – 26)

*(Focus: ASP.NET Core SignalR real-time WebSockets, Docker image packaging, Azure vs. AWS mapping, and Cloud deployment)*

#### Practical 22: Building a Real-Time WebSocket Hub using ASP.NET Core SignalR

* **Aim:** To create a `Notification.API` featuring an authenticated ASP.NET Core SignalR Hub for bi-directional real-time client communication.
* **Description:** Students create `Notification.API`, implement `OrderTrackingHub` decorated with `[Authorize]`, and configure `JwtBearerEvents.OnMessageReceived` to extract the JWT from the `?access_token=` query string during WebSocket handshakes.
* **Output Flow:**
1. Student opens a lightweight HTML/JS test client (or Postman WebSocket client) and connects to `ws://localhost:<port>/hubs/order-tracking?access_token=<JWT>`.
2. Server authenticates the user, maps `Context.UserIdentifier`, and logs `Client connected: user-111`.



#### Practical 23: Pushing Live RabbitMQ Saga Updates to Clients via SignalR

* **Aim:** To connect backend RabbitMQ order events to SignalR so users receive instant live browser notifications when their order is confirmed or cancelled.
* **Description:** Students add a MassTransit consumer in `Notification.API` listening to `StockReservedEvent` and `StockReservationFailedEvent`, injecting `IHubContext<OrderTrackingHub>` to call `Clients.User(userId).SendAsync("OrderStatusChanged", notificationDto)`.
* **Output Flow:**
1. Student connects User A and User B to the SignalR Hub in two separate browser tabs.
2. User A places an order via `Gateway.API`.
3. Within milliseconds of RabbitMQ processing, **only User A's browser tab** displays a real-time toast alert (`"Order #105 Confirmed!"`), while User B receives nothing.



#### Practical 24: Multi-Stage Containerization of Microservices using Dockerfiles

* **Aim:** To package all microservices (`Gateway.API`, `Identity.API`, `Catalog.API`, `Order.API`, `Notification.API`) into lightweight Linux Docker images using multi-stage Dockerfiles.
* **Description:** Students write multi-stage `Dockerfile`s (`[mcr.microsoft.com/dotnet/sdk](https://mcr.microsoft.com/dotnet/sdk)` for build $\rightarrow$ `[mcr.microsoft.com/dotnet/aspnet](https://mcr.microsoft.com/dotnet/aspnet)` for runtime) for each microservice and run the entire platform end-to-end strictly inside Docker containers.
* **Output Flow:**
1. Student runs `docker-compose up --build` to compile and containerize all 5 APIs plus infrastructure containers.
2. `docker images` shows optimized runtime images (~220MB each).
3. Student executes a full order flow against the containerized stack with zero Visual Studio/IDE processes running.



#### Practical 25: Cloud Architecture Design & Service Mapping (Azure vs. AWS)

* **Aim:** To analyze IaaS, PaaS, and SaaS cloud models and design a production cloud deployment architecture comparing Microsoft Azure and AWS managed services.
* **Description:** Students create an architectural mapping specification and environment configuration (`appsettings.Production.json`) mapping their local Docker containers to managed PaaS services:
* **Compute:** Azure Container Apps vs. AWS ECS (Fargate)
* **Database:** Azure Database for PostgreSQL vs. Amazon RDS for PostgreSQL
* **Cache:** Azure Cache for Redis vs. Amazon ElastiCache
* **Messaging:** Azure Service Bus vs. Amazon MQ (RabbitMQ)


* **Output Flow:**
1. Student produces a comparative architecture diagram and service-mapping table in the lab manual comparing cost, scaling, and management overhead across IaaS (EC2/Azure VM) vs. PaaS (ECS Fargate/Container Apps).
2. Student validates externalized environment variable injection (`ConnectionStrings__DefaultConnection`, `RabbitMq__Host`) locally using `.env` overrides.



#### Practical 26: Deploying Microservices to a Cloud Platform (Azure Container Apps / AWS ECS)

* **Aim:** To deploy the containerized microservices to a cloud container platform, connect managed cloud databases/caches, and verify end-to-end live operation.
* **Description:** Students push their Docker images to a container registry (Azure Container Registry / Docker Hub / Amazon ECR), provision managed PostgreSQL and Redis instances, deploy the Gateway and microservices to Azure Container Apps (or AWS ECS Fargate), and configure public ingress on `Gateway.API`.
* **Output Flow:**
1. Cloud portal displays all microservice containers in `Running` state with public HTTPS URLs assigned to `Gateway.API` and `Notification.API`.
2. Student executes the complete end-to-end workflow against the live cloud URL: Register $\rightarrow$ Login $\rightarrow$ Browse Cached Catalog $\rightarrow$ Place Idempotent Order $\rightarrow$ Receive Live SignalR Notification.
