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
### Practical 2: Developing `Catalog.API` with Entity Framework Core & SQL Server

#### 1. Aim & Task Description

To implement the Product Catalog microservice (`Catalog.API`) responsible for managing product inventory. Students will create the `Product` domain entity (`Id`, `Name`, `Price`, `StockQuantity`, `UpdatedAt`), configure `CatalogDbContext` with Entity Framework Core Code-First migrations connected directly to an isolated SQL Server database (`CatalogDb`), and build RESTful CRUD endpoints (`GET`, `POST`, `PUT`, `DELETE /api/products`).

---

#### 2. Prerequisites & Tools Required

* .NET 8.0 / 9.0 SDK
* Visual Studio 2022 or VS Code (with C# Dev Kit)
* SQL Server (LocalDB or SQL Server Express/Developer Edition) & SQL Server Management Studio (SSMS)
* Swagger UI or Postman for API testing
* Existing solution `ECommercePlatform.sln` with `Catalog.API` project (from Practical 1)

---

#### 3. Step-by-Step Implementation

##### Step 1: Install Required NuGet Packages

Open the terminal inside the `Catalog.API` project folder and install the Entity Framework Core SQL Server and Design packages:

```bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools

```

##### Step 2: Create the `Product` Entity

Create a folder named `Models` (or `Entities` in the Domain layer) and add `Product.cs`:

```csharp
namespace Catalog.API.Models;

public class Product
{
    public Guid Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int StockQuantity { get; set; }
    public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
}

```

##### Step 3: Create `CatalogDbContext`

Create a folder named `Data` and add `CatalogDbContext.cs` to map the `Product` entity to SQL Server:

```csharp
using Catalog.API.Models;
using Microsoft.EntityFrameworkCore;

namespace Catalog.API.Data;

public class CatalogDbContext : DbContext
{
    public CatalogDbContext(DbContextOptions<CatalogDbContext> options) : base(options)
    {
    }

    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>(entity =>
        {
            entity.HasKey(p => p.Id);
            entity.Property(p => p.Name).IsRequired().HasMaxLength(150);
            entity.Property(p => p.Price).HasColumnType("decimal(10,2)");
        });
    }
}

```

##### Step 4: Configure SQL Server Connection String

Open `appsettings.json` in `Catalog.API` and configure the isolated `CatalogDb` connection string:

```json
{
  "ConnectionStrings": {
    "CatalogConnection": "Server=(localdb)\\mssqllocaldb;Database=CatalogDb;Trusted_Connection=True;TrustServerCertificate=True;"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}

```

*(Note: If using SQL Server Express/Developer Edition instead of LocalDB, replace `(localdb)\\mssqllocaldb` with `localhost` or `localhost\\SQLEXPRESS`.)*

##### Step 5: Register `DbContext` and Build CRUD Endpoints

Open `Program.cs` and register `CatalogDbContext` along with the RESTful CRUD endpoints:

```csharp
using Catalog.API.Data;
using Catalog.API.Models;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Register EF Core with SQL Server (Database-per-Service)
builder.Services.AddDbContext<CatalogDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("CatalogConnection")));

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

// 1. GET All Products
app.MapGet("/api/products", async (CatalogDbContext db) =>
    await db.Products.AsNoTracking().ToListAsync());

// 2. GET Product by ID
app.MapGet("/api/products/{id:guid}", async (Guid id, CatalogDbContext db) =>
{
    var product = await db.Products.FindAsync(id);
    return product is not null ? Results.Ok(product) : Results.NotFound();
});

// 3. POST Create Product
app.MapPost("/api/products", async (Product input, CatalogDbContext db) =>
{
    var product = new Product
    {
        Id = Guid.NewGuid(),
        Name = input.Name,
        Price = input.Price,
        StockQuantity = input.StockQuantity,
        UpdatedAt = DateTime.UtcNow
    };

    db.Products.Add(product);
    await db.SaveChangesAsync();

    return Results.Created($"/api/products/{product.Id}", product);
});

// 4. PUT Update Product
app.MapPut("/api/products/{id:guid}", async (Guid id, Product input, CatalogDbContext db) =>
{
    var product = await db.Products.FindAsync(id);
    if (product is null) return Results.NotFound();

    product.Name = input.Name;
    product.Price = input.Price;
    product.StockQuantity = input.StockQuantity;
    product.UpdatedAt = DateTime.UtcNow;

    await db.SaveChangesAsync();
    return Results.Ok(product);
});

// 5. DELETE Product
app.MapDelete("/api/products/{id:guid}", async (Guid id, CatalogDbContext db) =>
{
    var product = await db.Products.FindAsync(id);
    if (product is null) return Results.NotFound();

    db.Products.Remove(product);
    await db.SaveChangesAsync();
    return Results.NoContent();
});

app.Run();

```

##### Step 6: Generate and Apply Code-First Migrations

Run the following commands in the terminal inside `Catalog.API` to create the `CatalogDb` database and `Products` table in SQL Server:

```bash
dotnet ef migrations add InitialCatalogCreate
dotnet ef database update

```

---

#### 4. Output & Verification Flow

Students must perform and document the following verification steps in their lab manual:

1. **Database Schema Verification (SSMS):**
* Connect to SQL Server in SSMS, expand `Databases -> CatalogDb -> Tables`, and confirm `dbo.Products` and `dbo.__EFMigrationsHistory` exist.


2. **Create Product (`POST /api/products`):**
* Execute a `POST` request via Swagger/Postman with JSON body:
```json
{
  "name": "Mechanical Keyboard",
  "price": 2500.00,
  "stockQuantity": 20
}

```


* **Expected Result:** HTTP `201 Created` with a newly generated `id` (GUID) and `updatedAt` timestamp.


3. **Read Products (`GET /api/products` & `GET /api/products/{id}`):**
* Call `GET /api/products` and confirm `"Mechanical Keyboard"` is returned with HTTP `200 OK`.


4. **Update Product (`PUT /api/products/{id}`):**
* Update `price` to `2200.00` and `stockQuantity` to `18` using the generated `id`.
* **Expected Result:** HTTP `200 OK` reflecting the updated values and refreshed `updatedAt` timestamp.


5. **Delete Product (`DELETE /api/products/{id}`):**
* Call `DELETE /api/products/{id}` and verify HTTP `204 No Content`. Subsequent `GET /api/products/{id}` calls return HTTP `404 Not Found`.



---

#### 5. Lab Viva / Review Questions

1. Why does `Catalog.API` use its own separate database (`CatalogDb`) instead of sharing a single database with `Order.API`?
2. What is the purpose of configuring `HasColumnType("decimal(10,2)")` on the `Price` property in `OnModelCreating`?
3. Why is `AsNoTracking()` used in the `GET /api/products` endpoint?
