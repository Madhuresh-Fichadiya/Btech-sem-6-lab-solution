### Project Theme: Distributed E-Commerce Order & Notification System

---

### Unit 1: Microservices & Design Architecture (Practicals 1 – 5)

*(Focus: Service decomposition, isolated SQL Server databases, and synchronous HTTP/gRPC communication)*

#### Practical 1: Scaffolding a Multi-Service Solution with Database-per-Service Architecture

* **Aim & Task Description:** Create a blank .NET solution (`ECommercePlatform.sln`) containing two independent ASP.NET Core Web API projects (`Catalog.API` and `Order.API`) structured using Clean Architecture layers (Domain, Application, Infrastructure, API). Configure separate SQL Server connection strings in each project's `appsettings.json` (`CatalogDb` and `OrderDb`) so each microservice maintains strict data sovereignty without sharing tables.
* **Output Flow:**
1. Student builds the solution (`dotnet build`) with zero compilation errors across both API projects.
2. Student inspects `appsettings.json` in both projects to verify distinct `Initial Catalog=CatalogDb` and `Initial Catalog=OrderDb` connection strings pointing to local SQL Server.
3. Running both projects simultaneously launches two separate Swagger UIs on different ports.



#### Practical 2: Developing `Catalog.API` with Entity Framework Core & SQL Server

* **Aim & Task Description:** Implement the Product Catalog microservice (`Catalog.API`) to manage product inventory. Create the `Product` entity (`Id`, `Name`, `Price`, `StockQuantity`, `UpdatedAt`), configure `CatalogDbContext` with EF Core Code-First migrations against SQL Server (`CatalogDb`), and build RESTful CRUD endpoints (`GET`, `POST`, `PUT`, `DELETE /api/products`).
* **Output Flow:**
1. Student runs `Update-Database` (or `dotnet ef database update`) and verifies the `Products` table inside SQL Server Management Studio (SSMS) under `CatalogDb`.
2. Student executes `POST /api/products` in Swagger/Postman to add `"Mechanical Keyboard"` (`Price: 2500`, `StockQuantity: 20`), receiving `201 Created`.
3. Student calls `GET /api/products` and verifies the stored record is retrieved from SQL Server.



#### Practical 3: Developing `Order.API` with Isolated Persistence

* **Aim & Task Description:** Implement the Order Management microservice (`Order.API`) backed by its own SQL Server database (`OrderDb`). Create `Order` and `OrderItem` entities storing `ProductId` as a standalone `Guid` (without cross-database foreign keys), apply EF Core migrations to create the tables in `OrderDb`, and expose `POST /api/orders` and `GET /api/orders/{id}` endpoints.
* **Output Flow:**
1. Student opens SSMS and confirms `Orders` and `OrderItems` tables exist inside `OrderDb` completely independent of `CatalogDb`.
2. Student sends a `POST /api/orders` request with product IDs and quantities.
3. `Order.API` persists the order with `Status = "Pending"` and returns `201 Created` along with the generated `OrderId`.



#### Practical 4: Synchronous Service-to-Service Communication using Typed `HttpClient`

* **Aim & Task Description:** Enable `Order.API` to validate live product prices and stock availability from `Catalog.API` before saving an order. Register a typed `ICatalogServiceClient` using `IHttpClientFactory` in `Order.API` that calls `GET /api/products/{id}` on `Catalog.API`, verifies `StockQuantity >= requestedQuantity`, calculates `TotalAmount` dynamically, and rejects orders with insufficient stock.
* **Output Flow:**
1. Student places an order via `POST /api/orders` for `2` units of a product priced at `2500`.
2. `Order.API` calls `Catalog.API` over HTTP, verifies stock (`20 >= 2`), computes `TotalAmount = 5000`, and returns `201 Created`.
3. Student attempts to order `50` units (exceeding stock) and verifies `Order.API` returns `400 Bad Request ("Insufficient stock")`.



#### Practical 5: High-Performance Inter-Service Communication using gRPC

* **Aim & Task Description:** Implement a binary gRPC service in `Catalog.API` for fast product price and stock verification, and compare its latency and payload size against REST/JSON. Define a `catalog.proto` contract (`CheckProductAvailability`), generate server/client C# stubs via `Grpc.AspNetCore`, host the gRPC endpoint in `Catalog.API`, and invoke it from `Order.API`.
* **Output Flow:**
1. Student triggers an order creation request routed through the gRPC client in `Order.API`.
2. `Catalog.API` logs an incoming `HTTP/2 application/grpc` call and returns the binary protobuf response.
3. Student compares REST JSON payload size (~340 bytes) vs. gRPC binary payload (~26 bytes) in their lab observation table.



---

### Unit 2: Caching, Rate Limiting, Logging and Security (Practicals 6 – 11)

*(Focus: Redis caching, Fixed Window vs. Token Bucket throttling, Serilog logging, and JWT Auth)*

#### Practical 6: Distributed Caching & Cache Invalidation using Redis

* **Aim & Task Description:** Improve read response times in `Catalog.API` and reduce SQL Server query load by integrating Redis (`StackExchangeRedisCache`). Run a Redis container on port `6379`, update `GET /api/products/{id}` to check Redis (`product:{id}`) before querying SQL Server (caching misses for 5 minutes), and invalidate the cache key whenever `PUT` or `DELETE` modifies a product.
* **Output Flow:**
1. First `GET /api/products/{id}` logs `[Cache MISS - Fetching from SQL Server]` (~40ms).
2. Immediate second `GET /api/products/{id}` logs `[Cache HIT - Fetching from Redis]` (<5ms).
3. Student updates the product price via `PUT /api/products/{id}`, calls `GET` again, and verifies a fresh cache miss occurs returning the updated price.



#### Practical 7: Implementing Fixed Window Rate Limiting on `Catalog.API`

* **Aim & Task Description:** Protect public product catalog endpoints against scraping and traffic spikes by configuring ASP.NET Core's `AddFixedWindowLimiter` middleware in `Catalog.API`. Partition requests by client IP address, permitting a maximum of `5 requests per 30 seconds` with `QueueLimit = 0`.
* **Output Flow:**
1. Student sends 5 rapid `GET /api/products` requests in Postman; all 5 succeed with `200 OK`.
2. On the 6th request within the 30-second window, `Catalog.API` blocks the request with `429 Too Many Requests` and includes a `Retry-After` header.
3. After 30 seconds elapse, requests succeed again.



#### Practical 8: Implementing Token Bucket Rate Limiting on `Order.API`

* **Aim & Task Description:** Control burst order submissions on `Order.API` using a Token Bucket rate limiter and compare its behavior with Fixed Window limiting. Configure `AddTokenBucketLimiter` on `POST /api/orders` with `TokenLimit = 4`, `TokensPerPeriod = 1`, and `ReplenishmentPeriod = 10 seconds` to allow short bursts while steadily refilling capacity.
* **Output Flow:**
1. Student sends 4 back-to-back `POST /api/orders` requests; all 4 succeed (`201 Created`).
2. The 5th immediate request fails with `429 Too Many Requests`.
3. Student waits 10 seconds (1 token refilled) and sends 1 request, verifying it succeeds immediately without waiting for a full window reset.



#### Practical 9: Structured Logging & Correlation ID Propagation with Serilog

* **Aim & Task Description:** Replace unstructured console logs with queryable JSON logs using `Serilog.AspNetCore` across `Catalog.API` and `Order.API`. Write custom middleware to read or generate an `X-Correlation-ID` header on every incoming request, push it to Serilog's `LogContext`, and forward it automatically on outgoing `HttpClient` requests.
* **Output Flow:**
1. Student sends `POST /api/orders` with header `X-Correlation-ID: btech-lab-01`.
2. `Order.API` writes structured JSON logs containing `"CorrelationId": "btech-lab-01"`.
3. `Catalog.API` receives the propagated header and logs `"CorrelationId": "btech-lab-01"`, linking logs across both services for a single request.



#### Practical 10: Building `Identity.API` for JWT Token Issuance

* **Aim & Task Description:** Develop a dedicated Authentication microservice (`Identity.API`) connected to its own SQL Server database (`IdentityDb`) to register users (`POST /api/auth/register`) with hashed passwords and authenticate logins (`POST /api/auth/login`) by issuing HMAC-SHA256 signed JSON Web Tokens (JWT) containing `sub` (UserId), `email`, and `role` (`Admin` or `Customer`) claims.
* **Output Flow:**
1. Student registers `admin@college.edu` (`Role: Admin`) and `student@college.edu` (`Role: Customer`) and verifies rows in `IdentityDb.Users`.
2. Student calls `POST /api/auth/login` and receives a signed JWT `access_token`.
3. Student inspects the decoded JWT payload to verify `sub`, `role`, and expiration (`exp`) claims.



#### Practical 11: Securing Microservice Endpoints with JWT & Role-Based Authorization

* **Aim & Task Description:** Configure `AddAuthentication().AddJwtBearer()` in both `Catalog.API` and `Order.API` using the shared JWT signing key. Enforce role-based access control by restricting `POST/PUT/DELETE /api/products` to `[Authorize(Roles = "Admin")]` and `POST /api/orders` to `[Authorize(Roles = "Customer")]`.
* **Output Flow:**
1. Calling `POST /api/products` without a token returns `401 Unauthorized`.
2. Calling `POST /api/products` with a `Customer` token returns `403 Forbidden`.
3. Calling `POST /api/products` with an `Admin` token succeeds (`201 Created`), and `Customer` successfully places an order on `POST /api/orders`.



---

### Unit 3: API Gateway, Resilience and Distributed Systems (Practicals 12 – 17)

*(Focus: Ocelot API Gateway, Polly Retry & Circuit Breaker, Redis Idempotency, Saga Pattern, and OpenTelemetry)*

#### Practical 12: Creating a Single Entry Point using Ocelot API Gateway

* **Aim & Task Description:** Build a dedicated `Gateway.API` project using the `Ocelot` NuGet package to hide internal microservice ports behind a single unified endpoint (`http://localhost:5000`). Configure `ocelot.json` to route `/gateway/catalog/{everything}`, `/gateway/orders/{everything}`, and `/gateway/auth/{everything}` to their respective services, and centralize JWT validation at the Gateway.
* **Output Flow:**
1. Student sends all Postman requests exclusively to `http://localhost:5000/gateway/...`.
2. Ocelot validates the JWT Bearer token at the edge and forwards valid requests to `Catalog.API` or `Order.API`.
3. Requests with missing/invalid tokens are rejected directly by Ocelot (`401 Unauthorized`) before reaching internal services.



#### Practical 13: Implementing Retry & Timeout Resilience Policies using Polly

* **Aim & Task Description:** Protect inter-service HTTP calls between `Order.API` and `Catalog.API` from transient network failures by attaching `AddStandardResilienceHandler()` (`Microsoft.Extensions.Http.Resilience` / Polly) to `ICatalogServiceClient`. Configure 3 exponential backoff retries (`200ms`, `400ms`, `800ms`) and a strict `2-second` per-attempt timeout.
* **Output Flow:**
1. Student injects a simulated `503 Service Unavailable` error on the first 2 calls inside `Catalog.API`.
2. Student sends `POST /gateway/orders`.
3. Console logs show Polly automatically retrying twice and succeeding on the 3rd attempt, returning `201 Created` seamlessly to the client.



#### Practical 14: Implementing Circuit Breaker & Graceful Degradation

* **Aim & Task Description:** Prevent `Order.API` from repeatedly calling a failing `Catalog.API` during extended outages by configuring a Polly Circuit Breaker policy that trips to the `Open` state for `20 seconds` when 50% of requests fail within 15 seconds. Catch `BrokenCircuitException` in `Order.API` to return a graceful fallback message.
* **Output Flow:**
1. Student stops `Catalog.API` and sends 5 order requests through `Gateway.API`.
2. Once the failure threshold trips, logs display `Circuit Breaker state changed to OPEN`.
3. Subsequent requests fail immediately in <2ms with `503 Service Unavailable ("Catalog service is currently unavailable. Please try again later.")`.



#### Practical 15: Handling Duplicate Requests using Redis Idempotency Keys

* **Aim & Task Description:** Prevent duplicate order creation when clients retry timed-out requests by implementing an `IdempotencyFilter` in `Order.API` backed by Redis. Require an `Idempotency-Key` header on `POST /api/orders`, check Redis (`idempotency:{userId}:{key}`), return the cached HTTP response if already processed, or execute and cache the response for 24 hours.
* **Output Flow:**
1. Student sends `POST /gateway/orders` with `Idempotency-Key: req-501`; `Order.API` creates Order `#1` in `OrderDb` (`201 Created`).
2. Student resends the identical request with `Idempotency-Key: req-501`.
3. `Order.API` returns the cached Order `#1` response from Redis without inserting a duplicate row in SQL Server.



#### Practical 16: Implementing Distributed Transactions via Choreography Saga

* **Aim & Task Description:** Maintain data consistency across `OrderDb` and `CatalogDb` without distributed two-phase commit locks by implementing an event-driven Choreography Saga. When `Order.API` creates a `Pending` order, it publishes `OrderCreatedEvent`; `Catalog.API` deducts stock in `CatalogDb` and publishes `StockReservedEvent` (or `StockReservationFailedEvent`), prompting `Order.API` to mark the order `Confirmed` or run a compensating update to `Cancelled`.
* **Output Flow:**
1. **Success Case:** Ordering `2` available units transitions `OrderDb.Status` from `Pending` $\rightarrow$ `Confirmed` and decrements `CatalogDb.StockQuantity` by `2`.
2. **Compensation Case:** Ordering `999` units (out of stock) triggers `StockReservationFailedEvent`, causing `Order.API` to automatically roll back `OrderDb.Status` to `Cancelled`.



#### Practical 17: End-to-End Distributed Tracing with OpenTelemetry and Jaeger

* **Aim & Task Description:** Instrument `Gateway.API`, `Order.API`, and `Catalog.API` with OpenTelemetry (`OpenTelemetry.Extensions.Hosting`) to capture incoming HTTP, outgoing `HttpClient`, and SQL Server EF Core spans, and export them to a Jaeger container (`http://localhost:16686`) for end-to-end transaction visualization.
* **Output Flow:**
1. Student submits an order through `Gateway.API`.
2. Student opens Jaeger UI (`http://localhost:16686`), selects the trace by `TraceId`, and inspects the unified waterfall diagram displaying exact execution durations across Gateway $\rightarrow$ Order.API $\rightarrow$ Redis $\rightarrow$ Catalog.API $\rightarrow$ SQL Server.



---

### Unit 4: Background Jobs, Queuing & Messaging (Practicals 18 – 21)

*(Focus: RabbitMQ Producer–Consumer architecture, Dead-Letter Queues, and Hangfire scheduling)*

#### Practical 18: Asynchronous Producer–Consumer Messaging with RabbitMQ & MassTransit

* **Aim & Task Description:** Configure RabbitMQ (running on ports `5672` and `15672`) with MassTransit across `Order.API` and `Catalog.API` to decouple microservices using durable exchanges, queues, producers, and consumers for reliable asynchronous event delivery.
* **Output Flow:**
1. Student inspects the RabbitMQ Management Portal (`http://localhost:15672`) to verify auto-configured durable exchanges and queues.
2. Student stops `Catalog.API`, places 3 orders in `Order.API`, and observes `3 Ready` messages safely queued in RabbitMQ.
3. Student restarts `Catalog.API` and observes all 3 messages immediately consumed (`0 Ready`).



#### Practical 19: Fault-Tolerant Message Consumption & Dead-Letter Queues (DLQ)

* **Aim & Task Description:** Protect asynchronous consumers against poison messages and transient SQL Server deadlocks by configuring MassTransit retry policies (`UseMessageRetry` with 3 intervals) and Dead-Letter Queues (`*_error`) so failing messages are isolated without blocking healthy queue traffic.
* **Output Flow:**
1. Student publishes a simulated poison event that throws an unhandled exception inside the consumer.
2. Logs show MassTransit retrying the message 3 times before RabbitMQ automatically routes it to the `order-created_error` Dead-Letter Queue.
3. Student verifies in RabbitMQ UI that the poison message sits in the DLQ while subsequent valid orders process normally.



#### Practical 20: Background Job Processing & Fire-and-Forget Jobs using Hangfire

* **Aim & Task Description:** Offload slow post-order tasks (generating a text/PDF order invoice) from the checkout API response path by configuring `Hangfire.AspNetCore` with SQL Server storage (`Hangfire.SqlServer`) in `Order.API`. Enqueue a Fire-and-Forget job (`BackgroundJob.Enqueue`) when an order is confirmed to generate the invoice file and update `Orders.InvoiceUrl`.
* **Output Flow:**
1. Student opens `http://localhost:<port>/hangfire` to view the Hangfire Dashboard and auto-created `HangFire.*` tables in `OrderDb`.
2. Upon order confirmation, `Order.API` enqueues the invoice job without delaying the API response.
3. Student verifies the job under `Succeeded Jobs` in the Hangfire Dashboard and confirms `InvoiceUrl` is saved in SQL Server.



#### Practical 21: Automating Order Expiration & Reports using Hangfire Delayed & Recurring Jobs

* **Aim & Task Description:** Implement time-based background automation in `Order.API` using Hangfire: schedule a Delayed Job (`BackgroundJob.Schedule(..., TimeSpan.FromMinutes(2))`) upon order creation that cancels any order still in `Pending` state, and register a CRON Recurring Job (`RecurringJob.AddOrUpdate`) to compute daily sales totals.
* **Output Flow:**
1. Student places an order while `Catalog.API` is offline so it remains `Pending`.
2. After 2 minutes, the Delayed Job triggers automatically and updates the order status to `Cancelled` in `OrderDb`.
3. Student triggers the `daily-sales-summary` Recurring Job from the Hangfire Dashboard and verifies the total revenue calculation in the console output.



---

### Unit 5: Cloud & Realtime Communication (Practicals 22 – 26)

*(Focus: ASP.NET Core SignalR WebSockets, Docker containerization, Azure vs. AWS mapping, and Cloud deployment)*

#### Practical 22: Building an Authenticated Real-Time Hub using ASP.NET Core SignalR

* **Aim & Task Description:** Create a `Notification.API` microservice containing an `[Authorize]` SignalR Hub (`OrderTrackingHub`) for real-time WebSocket communication. Configure `JwtBearerEvents.OnMessageReceived` to extract the JWT from the `?access_token=` query string during WebSocket handshakes and map users via `Context.UserIdentifier`.
* **Output Flow:**
1. Student connects a lightweight HTML/JavaScript client (or Postman WebSocket request) to `ws://localhost:<port>/hubs/order-tracking?access_token=<JWT>`.
2. `Notification.API` validates the JWT during handshake and logs `User connected: <UserId>`.



#### Practical 23: Pushing Live RabbitMQ Saga Updates to Clients via SignalR

* **Aim & Task Description:** Bridge backend RabbitMQ Saga events to connected browser clients in real time. Add MassTransit consumers in `Notification.API` for `StockReservedEvent` and `StockReservationFailedEvent` that inject `IHubContext<OrderTrackingHub>` and invoke `Clients.User(userId).SendAsync("OrderStatusUpdated", payload)`.
* **Output Flow:**
1. Student opens two browser tabs logged in as `User A` and `User B` connected to `OrderTrackingHub`.
2. `User A` submits an order through `Gateway.API`.
3. Immediately after RabbitMQ processes the stock reservation, **only `User A`'s browser tab** displays a live notification (`"Order Confirmed!"`), while `User B` receives nothing.



#### Practical 24: Containerizing Microservices using Multi-Stage Dockerfiles

* **Aim & Task Description:** Package all five ASP.NET Core microservices (`Gateway.API`, `Identity.API`, `Catalog.API`, `Order.API`, `Notification.API`) into lightweight Linux container images using multi-stage `Dockerfile`s (`dotnet/sdk` build stage $\rightarrow$ `dotnet/aspnet` runtime stage) and externalize SQL Server, Redis, and RabbitMQ connection strings via environment variables.
* **Output Flow:**
1. Student builds all Docker images (`docker build -t catalog-api .`, etc.) and lists optimized ~220MB runtime images using `docker images`.
2. Student runs the containerized microservices connected to SQL Server and verifies full API functionality without running Visual Studio.



#### Practical 25: Cloud Architecture Design & Service Mapping (Azure vs. AWS)

* **Aim & Task Description:** Compare IaaS, PaaS, and SaaS cloud service models and design a cloud deployment blueprint mapping local lab components to managed PaaS offerings across Microsoft Azure and AWS:
* **Compute:** Azure Container Apps vs. AWS ECS (Fargate)
* **Database:** Azure SQL Database vs. Amazon RDS for SQL Server
* **Cache:** Azure Cache for Redis vs. Amazon ElastiCache
* **Messaging:** Azure Service Bus vs. Amazon MQ (RabbitMQ)


* **Output Flow:**
1. Student submits an architectural block diagram and comparison table in the lab manual contrasting scaling, pricing model, and operational responsibility between IaaS (VMs) and PaaS (Container Apps / Fargate).
2. Student validates externalized `.env` configuration overrides locally before cloud deployment.



#### Practical 26: Deploying Microservices to the Cloud (Azure Container Apps / AWS ECS)

* **Aim & Task Description:** Push the containerized microservice images to a container registry (Docker Hub / Azure Container Registry / Amazon ECR), provision managed cloud SQL Server and Redis instances, deploy the Gateway and microservices to Azure Container Apps (or AWS ECS Fargate), and execute a live end-to-end test.
* **Output Flow:**
1. Cloud dashboard displays all microservice containers in `Running` status with public HTTPS endpoints exposed on `Gateway.API` and `Notification.API`.
2. Student runs the complete live workflow against the cloud URL: Register $\rightarrow$ Login $\rightarrow$ Browse Cached Catalog $\rightarrow$ Place Idempotent Order $\rightarrow$ Receive Real-Time SignalR Notification.
---
---
---
Here are all **26 practicals** written in your exact format (**Title**, **Aim**, and **Task Description** only—no code solutions), aligned to the **E-Commerce Order & Notification System** using direct **SQL Server** connections and strict unit boundaries.

---

### Unit 1: Microservices & Design Architecture (Practicals 1 – 5)

#### Practical 1: Scaffolding a Multi-Service Solution with Database-per-Service Architecture

* **Aim:** To scaffold a multi-service ASP.NET Core Web API solution (`Catalog.API` and `Order.API`) following Clean Architecture and the Database-per-Service pattern using SQL Server.
* **Task Description:** Create a blank .NET solution (`ECommercePlatform.sln`) containing two independent Web API projects (`Catalog.API` and `Order.API`) organized into Domain, Application, Infrastructure, and API layers. Configure separate SQL Server connection strings in each project's `appsettings.json` (`CatalogDb` and `OrderDb`) so each microservice maintains isolated persistence without sharing tables or foreign keys.

#### Practical 2: Developing `Catalog.API` with Entity Framework Core & SQL Server

* **Aim:** To implement the Product Catalog microservice (`Catalog.API`) with its own SQL Server database (`CatalogDb`) using Entity Framework Core Code-First migrations and expose CRUD endpoints.
* **Task Description:** Create the `Product` entity (`Id`, `Name`, `Price`, `StockQuantity`, `UpdatedAt`), configure `CatalogDbContext` with EF Core Code-First migrations against SQL Server (`CatalogDb`), and build RESTful CRUD endpoints (`GET`, `POST`, `PUT`, `DELETE /api/products`) to manage product catalog inventory.

#### Practical 3: Developing `Order.API` with Isolated Persistence

* **Aim:** To implement the Order Management microservice (`Order.API`) backed by its own isolated SQL Server database (`OrderDb`) without cross-database dependencies.
* **Task Description:** Create the `Order` entity (`Id`, `UserId`, `ProductId`, `ProductName`, `Quantity`, `TotalAmount`, `Status`, `InvoiceUrl`, `CreatedAt`) storing `ProductId` and `UserId` as standalone `Guid` values without cross-database foreign keys. Apply EF Core migrations to generate the `Orders` table in `OrderDb` and expose `POST /api/orders` and `GET /api/orders/{id}` endpoints.

#### Practical 4: Synchronous Service-to-Service Communication using Typed `HttpClient`

* **Aim:** To implement synchronous inter-service communication between `Order.API` and `Catalog.API` using a typed `HttpClient` so `Order.API` can validate real-time product pricing and stock availability before creating an order.
* **Task Description:** Register a typed `ICatalogServiceClient` via `IHttpClientFactory` in `Order.API` configured with `Catalog.API`'s base URL. Inside `POST /api/orders`, invoke `GET /api/products/{id}` on `Catalog.API` to verify that the requested product exists (`200 OK`) and has sufficient inventory (`StockQuantity >= RequestedQuantity`). Calculate `TotalAmount` dynamically using the live price returned by `Catalog.API`, save the order as `Pending` in `OrderDb`, and reject requests with `404 Not Found` or `400 Bad Request` when the product is missing or out of stock.

#### Practical 5: High-Performance Inter-Service Communication using gRPC

* **Aim:** To implement a binary gRPC endpoint in `Catalog.API` for fast product availability checks and compare its payload size and latency against REST/JSON.
* **Task Description:** Define a `catalog.proto` contract (`CheckProductAvailability`), generate C# server and client stubs using `Grpc.AspNetCore`, host the gRPC service inside `Catalog.API`, and configure a gRPC client in `Order.API` to fetch product price and stock over HTTP/2 Protobuf.

---

### Unit 2: Caching, Rate Limiting, Logging and Security (Practicals 6 – 11)

#### Practical 6: Distributed Read-Through Caching & Cache Invalidation with Redis

* **Aim:** To reduce SQL Server query load and accelerate product read operations in `Catalog.API` using Redis distributed caching and cache invalidation.
* **Task Description:** Run a Redis instance on port `6379` and register `AddStackExchangeRedisCache` in `Catalog.API`. Update `GET /api/products/{id}` to check Redis (`product:{id}`) before querying SQL Server, caching misses with a 5-minute expiration, and explicitly remove the Redis cache key whenever `PUT` or `DELETE` modifies or deletes a product.

#### Practical 7: Implementing Fixed Window Rate Limiting on `Catalog.API`

* **Aim:** To protect public product browsing endpoints in `Catalog.API` from scraping and traffic spikes using the Fixed Window rate limiting algorithm.
* **Task Description:** Configure ASP.NET Core's `AddFixedWindowLimiter` middleware in `Catalog.API` partitioned by client IP address, permitting a maximum of `5 requests per 30 seconds` with `QueueLimit = 0`, and return HTTP `429 Too Many Requests` with a `Retry-After` header when the limit is exceeded.

#### Practical 8: Implementing Token Bucket Rate Limiting on `Order.API`

* **Aim:** To control order submission bursts in `Order.API` using the Token Bucket rate limiting algorithm and compare its traffic-smoothing behavior against Fixed Window limiting.
* **Task Description:** Configure `AddTokenBucketLimiter` on `POST /api/orders` with `TokenLimit = 4`, `TokensPerPeriod = 1`, and `ReplenishmentPeriod = 10 seconds` to allow short bursts of order submissions while gradually replenishing capacity over time.

#### Practical 9: Structured Logging & Correlation ID Propagation with Serilog

* **Aim:** To replace plain-text logs with structured JSON logging using Serilog and trace requests across microservices via an `X-Correlation-ID` header.
* **Task Description:** Configure `Serilog.AspNetCore` in both `Catalog.API` and `Order.API`. Implement custom middleware that reads or generates an `X-Correlation-ID` header on incoming requests, pushes it into Serilog's `LogContext`, and forwards the header automatically via a `DelegatingHandler` on outgoing `HttpClient` calls.

#### Practical 10: Building `Identity.API` for JWT Token Issuance

* **Aim:** To implement a dedicated Authentication microservice (`Identity.API`) connected to SQL Server (`IdentityDb`) that authenticates users and issues signed JSON Web Tokens (JWT).
* **Task Description:** Create `Identity.API` with a `Users` table (`Id`, `FullName`, `Email`, `PasswordHash`, `Role`) in `IdentityDb`. Implement `POST /api/auth/register` with secure password hashing and `POST /api/auth/login` to generate HMAC-SHA256 signed JWT access tokens embedding `sub` (UserId), `email`, and `role` (`Admin` or `Customer`) claims.

#### Practical 11: Securing Microservice Endpoints with JWT & Role-Based Authorization

* **Aim:** To protect `Catalog.API` and `Order.API` endpoints using JWT Bearer authentication and enforce role-based authorization policies (`Admin` vs. `Customer`).
* **Task Description:** Configure `AddAuthentication().AddJwtBearer()` in both `Catalog.API` and `Order.API` using the shared JWT signing key. Restrict product creation, updates, and deletion (`POST`, `PUT`, `DELETE /api/products`) to `[Authorize(Roles = "Admin")]` and restrict order placement (`POST /api/orders`) to `[Authorize(Roles = "Customer")]`, extracting `UserId` directly from the authenticated JWT claims.

---

### Unit 3: API Gateway, Resilience and Distributed Systems (Practicals 12 – 17)

#### Practical 12: Creating a Single Entry Point using Ocelot API Gateway

* **Aim:** To hide internal microservice ports behind a single unified entry point (`Gateway.API`) using Ocelot and centralize upstream routing and JWT validation.
* **Task Description:** Create a `Gateway.API` project using the `Ocelot` package, configure `ocelot.json` to map upstream routes (`/gateway/catalog/{everything}`, `/gateway/orders/{everything}`, `/gateway/auth/{everything}`) to their downstream microservice ports, and enforce JWT Bearer validation at the Gateway layer.

#### Practical 13: Implementing Retry & Timeout Resilience Policies using Polly

* **Aim:** To make synchronous HTTP calls between `Order.API` and `Catalog.API` resilient against transient network failures using Exponential Backoff Retries and Timeouts.
* **Task Description:** Attach `AddStandardResilienceHandler()` (`Microsoft.Extensions.Http.Resilience` / Polly) to `ICatalogServiceClient` in `Order.API`. Configure 3 exponential backoff retry attempts (`200ms`, `400ms`, `800ms`) for transient `5xx` or timeout failures along with a strict `2-second` per-attempt timeout.

#### Practical 14: Implementing Circuit Breaker & Graceful Degradation

* **Aim:** To prevent cascading failures when `Catalog.API` experiences prolonged downtime by configuring a Polly Circuit Breaker policy in `Order.API`.
* **Task Description:** Configure a Circuit Breaker strategy on `ICatalogServiceClient` that trips to the `Open` state for `20 seconds` if 50% of requests fail within a 15-second sampling window. Catch `BrokenCircuitException` in `Order.API` to immediately return a graceful HTTP `503 Service Unavailable` fallback response without waiting for network timeouts.

#### Practical 15: Handling Duplicate Requests using Redis Idempotency Keys

* **Aim:** To prevent duplicate order creation when clients retry timed-out requests by implementing an Idempotency filter in `Order.API` backed by Redis.
* **Task Description:** Build an `IdempotencyFilter` in `Order.API` that requires an `Idempotency-Key` header on `POST /api/orders`. Check Redis (`idempotency:{userId}:{key}`) before executing the handler; if found, return the cached HTTP status code and JSON response immediately, otherwise execute the order creation and cache the result in Redis for 24 hours.

#### Practical 16: Implementing Distributed Transactions via Choreography Saga

* **Aim:** To maintain eventual data consistency across `OrderDb` and `CatalogDb` without distributed two-phase commit locks using an event-driven Choreography Saga.
* **Task Description:** Refactor order fulfillment into a Choreography Saga where `Order.API` saves the order in `Pending` state and publishes `OrderCreatedEvent`. `Catalog.API` consumes the event, deducts `StockQuantity` in `CatalogDb`, and publishes either `StockReservedEvent` or `StockReservationFailedEvent`, prompting `Order.API` to transition the order to `Confirmed` or run a compensating update to `Cancelled`.

#### Practical 17: End-to-End Distributed Tracing with OpenTelemetry and Jaeger

* **Aim:** To trace a single distributed request across Ocelot, HTTP calls, SQL Server, Redis, and message queues using OpenTelemetry and Jaeger.
* **Task Description:** Run a Jaeger container (`http://localhost:16686`) and instrument `Gateway.API`, `Order.API`, and `Catalog.API` with OpenTelemetry (`OpenTelemetry.Extensions.Hosting`) to capture ASP.NET Core, `HttpClient`, and EF Core trace spans, exporting them via OTLP to visualize the unified waterfall timeline.

---

### Unit 4: Background Jobs, Queuing & Messaging (Practicals 18 – 21)

#### Practical 18: Asynchronous Producer–Consumer Messaging with RabbitMQ & MassTransit

* **Aim:** To decouple microservices using asynchronous message-oriented communication with RabbitMQ and MassTransit.
* **Task Description:** Configure RabbitMQ (`ports 5672 / 15672`) with MassTransit across `Order.API` and `Catalog.API`. Define durable exchanges, queues, producers, and consumers for order and inventory events so messages are buffered safely in RabbitMQ even when a consumer service is temporarily offline.

#### Practical 19: Fault-Tolerant Message Consumption & Dead-Letter Queues (DLQ)

* **Aim:** To handle poison messages and transient database errors during queue consumption using retry policies and Dead-Letter Queues in RabbitMQ.
* **Task Description:** Configure `UseMessageRetry` with progressive intervals on MassTransit consumers in `Catalog.API` and `Order.API`. Simulate an unhandled consumer exception to verify that failed messages are retried and subsequently routed to RabbitMQ's `*_error` Dead-Letter Queue without blocking valid messages.

#### Practical 20: Background Job Processing & Fire-and-Forget Jobs using Hangfire

* **Aim:** To offload non-blocking post-order invoice generation from the checkout API response path using Hangfire backed by SQL Server.
* **Task Description:** Install `Hangfire.AspNetCore` and `Hangfire.SqlServer` in `Order.API`, expose the `/hangfire` monitoring dashboard, and enqueue a Fire-and-Forget background job (`BackgroundJob.Enqueue`) upon order confirmation to generate an invoice file and update `Orders.InvoiceUrl` in `OrderDb`.

#### Practical 21: Automating Order Expiration & Reports using Hangfire Delayed & Recurring Jobs

* **Aim:** To automate unpaid order cancellation using Hangfire Delayed Jobs and daily revenue reporting using CRON Recurring Jobs.
* **Task Description:** Schedule a Hangfire Delayed Job (`BackgroundJob.Schedule`) when an order is created in `Pending` state that automatically marks the order `Cancelled` if it remains unconfirmed after a set timeout, and register a daily CRON Recurring Job (`RecurringJob.AddOrUpdate`) that aggregates confirmed daily sales totals.

---

### Unit 5: Cloud & Realtime Communication (Practicals 22 – 26)

#### Practical 22: Building an Authenticated Real-Time Hub using ASP.NET Core SignalR

* **Aim:** To build a `Notification.API` microservice featuring an authenticated ASP.NET Core SignalR Hub for real-time WebSocket client connections.
* **Task Description:** Create `Notification.API` containing an `[Authorize]` SignalR Hub (`OrderTrackingHub`). Configure `JwtBearerEvents.OnMessageReceived` to extract the JWT access token from the `?access_token=` query string during WebSocket handshakes so connected clients are mapped automatically via `Context.UserIdentifier`.

#### Practical 23: Pushing Live RabbitMQ Saga Updates to Clients via SignalR

* **Aim:** To connect backend RabbitMQ Saga events to SignalR so users receive instant live browser notifications when their order status changes.
* **Task Description:** Add MassTransit consumers in `Notification.API` listening to `StockReservedEvent` and `StockReservationFailedEvent`. Inject `IHubContext<OrderTrackingHub>` into the consumers and invoke `Clients.User(userId).SendAsync("OrderStatusUpdated", payload)` to deliver real-time confirmation or cancellation alerts exclusively to the ordering user.

#### Practical 24: Containerizing Microservices using Multi-Stage Dockerfiles

* **Aim:** To package all microservices into lightweight Linux container images using multi-stage Dockerfiles and externalized environment configurations.
* **Task Description:** Write multi-stage `Dockerfile`s (`[mcr.microsoft.com/dotnet/sdk](https://mcr.microsoft.com/dotnet/sdk)` build stage $\rightarrow$ `[mcr.microsoft.com/dotnet/aspnet](https://mcr.microsoft.com/dotnet/aspnet)` runtime stage) for `Gateway.API`, `Identity.API`, `Catalog.API`, `Order.API`, and `Notification.API`, externalizing all SQL Server, Redis, RabbitMQ, and JWT settings via environment variables.

#### Practical 25: Cloud Architecture Design & Service Mapping (Azure vs. AWS)

* **Aim:** To analyze IaaS, PaaS, and SaaS cloud service models and design a cloud deployment blueprint mapping local lab components to managed PaaS offerings on Microsoft Azure and AWS.
* **Task Description:** Prepare a cloud architecture specification comparing IaaS virtual machines against managed PaaS services across Azure and AWS—mapping containers to Azure Container Apps / AWS ECS Fargate, databases to Azure SQL Database / Amazon RDS for SQL Server, caching to Azure Cache for Redis / Amazon ElastiCache, and messaging to Azure Service Bus / Amazon MQ—and validate `.env` configuration overrides locally.

#### Practical 26: Deploying Microservices to the Cloud (Azure Container Apps / AWS ECS)

* **Aim:** To deploy the containerized microservices to a cloud container platform, connect managed cloud database and cache instances, and verify live end-to-end execution.
* **Task Description:** Push the microservice container images to a container registry (Docker Hub / Azure Container Registry / Amazon ECR), provision managed cloud SQL Server and Redis instances, deploy the Gateway and backend microservices to Azure Container Apps (or AWS ECS Fargate), and execute an end-to-end test verifying authentication, cached browsing, idempotent ordering, and live SignalR notifications over public HTTPS endpoints.
