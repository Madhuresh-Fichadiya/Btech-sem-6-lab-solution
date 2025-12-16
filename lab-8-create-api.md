# Lab 8: Create API – I (Roles API & User Management API)

This lab teaches you how to build RESTful APIs using ASP.NET Core Web API with Entity Framework Core. You'll learn HTTP verbs, routing, and create APIs for managing Roles and Users.

## What You'll Learn

- Understanding HTTP verbs (GET, POST, PUT, DELETE)
- How to use route attributes in controllers
- How to use HTTP verb attributes
- Building CRUD APIs for Roles
- Building CRUD APIs for Users

---

## Task 1: Understanding HTTP Verbs and Routing

### What Are HTTP Verbs?

HTTP verbs (also called HTTP methods) tell the server what action you want to perform on a resource. Think of them as commands you send to the server.

| HTTP Verb  | What It Does                     | Real-World Example              | Database Operation    |
| ---------- | -------------------------------- | ------------------------------- | --------------------- |
| **GET**    | Gets/reads data from server      | View a list of products         | SELECT (read)         |
| **POST**   | Creates new data on server       | Add a new product               | INSERT (create)       |
| **PUT**    | Updates existing data completely | Update all details of a product | UPDATE (edit)         |
| **DELETE** | Removes data from server         | Delete a product                | DELETE (remove)       |
| **PATCH**  | Updates part of existing data    | Update only product price       | UPDATE (partial edit) |

**Simple Analogy:**

- **GET** = Reading a book (you don't change anything)
- **POST** = Writing a new book (creating something new)
- **PUT** = Rewriting an entire book (replacing everything)
- **DELETE** = Throwing away a book (removing it)

### HTTP Verbs in Real Life

Imagine a library system:

```
GET    /books          → Show me all books
GET    /books/5        → Show me book with ID 5
POST   /books          → Add a new book to the library
PUT    /books/5        → Update all details of book 5
DELETE /books/5        → Remove book 5 from library
```

### Why Use Different HTTP Verbs?

Without HTTP verbs, you'd need separate URLs for everything:

```
❌ Bad approach (without HTTP verbs):
/books/getall
/books/getbyid/5
/books/create
/books/update/5
/books/delete/5

✅ Good approach (with HTTP verbs):
GET    /books
GET    /books/5
POST   /books
PUT    /books/5
DELETE /books/5
```

---

## How HTTP Verbs Are Defined in Controllers

In ASP.NET Core, you use **attributes** to define which HTTP verb each method responds to.

### HTTP Verb Attributes

| Attribute      | HTTP Verb | Common Use                         | Method Syntax                                                    |
| -------------- | --------- | ---------------------------------- | ---------------------------------------------------------------- |
| `[HttpGet]`    | GET       | Read/retrieve data                 | `[HttpGet]`<br/>`public IActionResult GetAll()`                  |
| `[HttpPost]`   | POST      | Create new data                    | `[HttpPost]`<br/>`public IActionResult Create()`                 |
| `[HttpPut]`    | PUT       | Update existing data (full update) | `[HttpPut("{id}")]`<br/>`public IActionResult Update(int id)`    |
| `[HttpDelete]` | DELETE    | Delete data                        | `[HttpDelete("{id}")]`<br/>`public IActionResult Delete(int id)` |
| `[HttpPatch]`  | PATCH     | Partial update of data             | `[HttpPatch("{id}")]`<br/>`public IActionResult Patch(int id)`   |

### Basic Controller Structure

```csharp
[ApiController]                      // Marks this as an API controller
[Route("api/[controller]")]          // Sets the base route
public class RolesController : ControllerBase
{
    // Your methods go here
}
```

---

## Understanding Route Attributes

Routes determine what URL path triggers which controller method.

### 1. Controller-Level Route Attribute

The `[Route]` attribute on the controller sets the **base path** for all methods in that controller.

```csharp
[ApiController]
[Route("api/[controller]")]          // Base route
public class RolesController : ControllerBase
{
    // All methods inherit "api/roles" as base path
}
```

**What `[controller]` means:**

- `[controller]` is a placeholder that gets replaced with the controller name (without "Controller")
- `RolesController` → route becomes `api/roles`
- `UsersController` → route becomes `api/users`

**You can also use a fixed route:**

```csharp
[Route("api/roles")]                 // Fixed route (no placeholder)
public class RolesController : ControllerBase
{
    // Methods are under "api/roles"
}
```

### 2. Method-Level Route Attributes

You can add more to the route for specific methods:

| Route Pattern | Attribute                   | Final URL             | Explanation                |
| ------------- | --------------------------- | --------------------- | -------------------------- |
| Base only     | `[HttpGet]`                 | `api/roles`           | Uses only controller route |
| With ID       | `[HttpGet("{id}")]`         | `api/roles/5`         | Adds ID parameter to route |
| With text     | `[HttpGet("search")]`       | `api/roles/search`    | Adds fixed text to route   |
| Combined      | `[HttpGet("{id}/details")]` | `api/roles/5/details` | Combines ID and text       |

### Understanding Route Parameters

**`{id}` in routes:**

- `{id}` is a **route parameter** (placeholder for dynamic values)
- The value from the URL is automatically passed to your method parameter
- Must match the parameter name (case-insensitive)

**Example:**

```csharp
[HttpGet("{id}")]
public IActionResult GetById(int id)
```

If URL is `api/roles/5`, then `id = 5`

### Route Parameter Types

You can specify the type of parameter expected:

| Constraint    | Example             | What It Matches      | Usage                            |
| ------------- | ------------------- | -------------------- | -------------------------------- |
| `:int`        | `{id:int}`          | Integer numbers only | `[HttpGet("{id:int}")]`          |
| `:alpha`      | `{name:alpha}`      | Letters only         | `[HttpGet("{name:alpha}")]`      |
| `:min(value)` | `{id:int:min(1)}`   | Minimum value        | `[HttpGet("{id:int:min(1)}")]`   |
| `:max(value)` | `{id:int:max(100)}` | Maximum value        | `[HttpGet("{id:int:max(100)}")]` |
| `:length(n)`  | `{code:length(5)}`  | Exact length         | `[HttpGet("{code:length(5)}")]`  |

---

## Combining Route and HTTP Verb Attributes

You can combine routes directly in HTTP verb attributes:

### Method 1: Separate Attributes (More Clear)

```csharp
[HttpGet]
[Route("{id}")]
public IActionResult GetById(int id) { }
```

### Method 2: Combined (More Compact)

```csharp
[HttpGet("{id}")]
public IActionResult GetById(int id) { }
```

Both methods work the same way. Use whichever is clearer for you.

---

## Understanding Parameter Binding Attributes

These attributes tell ASP.NET where to get the data from:

| Attribute      | Where Data Comes From    | Example URL/Data                      | Method Usage                             |
| -------------- | ------------------------ | ------------------------------------- | ---------------------------------------- |
| `[FromRoute]`  | From URL path            | `api/roles/5` → id=5                  | `GetById([FromRoute] int id)`            |
| `[FromQuery]`  | From URL query string    | `api/roles?name=admin` → name="admin" | `Search([FromQuery] string name)`        |
| `[FromBody]`   | From request body (JSON) | `{ "name": "Admin" }`                 | `Create([FromBody] Role role)`           |
| `[FromHeader]` | From HTTP headers        | `Authorization: Bearer token`         | `Get([FromHeader] string authorization)` |
| `[FromForm]`   | From form data           | Form submission                       | `Upload([FromForm] IFormFile file)`      |

**Note:** Most of the time, ASP.NET automatically knows where to get data from, so these attributes are optional. Use them when you need to be explicit.

---

## Common Errors Possibilities

### Error 1: Missing HTTP Verb Attribute

**Problem:** Method without an HTTP verb attribute won't be accessible as an API endpoint.

❌ **Example:**

```csharp
[Route("api/[controller]")]
public class RolesController : ControllerBase
{
    // Missing HTTP verb attribute - this won't work!
    public IActionResult GetAll()
    {
        return Ok("roles");
    }
}
```

**Error Message:**

```
HTTP Error 404 - Not Found
```

---

### Error 2: Route Conflicts - Same HTTP Verb with Same Route

**Problem:** Multiple methods with the same HTTP verb and same route pattern cause conflicts.

❌ **Example:**

```csharp
[Route("api/[controller]")]
public class RolesController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult GetById(int id)
    {
        return Ok($"Role {id}");
    }

    [HttpGet("{roleId}")]  // Conflict! Same as above
    public IActionResult GetRoleDetails(int roleId)
    {
        return Ok($"Details {roleId}");
    }
}
```

**Error Message:**

```
AmbiguousMatchException: The request matched multiple endpoints.
```

---

### Error 3: Multiple Same HTTP Verbs Without Route Differentiation

**Problem:** Two or more methods with same HTTP verb on base route.

❌ **Example:**

```csharp
[Route("api/[controller]")]
public class RolesController : ControllerBase
{
    [HttpGet]  // Route: api/roles
    public IActionResult GetAll()
    {
        return Ok("All roles");
    }

    [HttpGet]  // Route: api/roles - CONFLICT!
    public IActionResult GetActive()
    {
        return Ok("Active roles");
    }
}
```

**Error Message:**

```
AmbiguousMatchException: The request matched multiple endpoints
```

---

### Error 4: Parameter Name Mismatch

**Problem:** Route parameter name doesn't match method parameter name.

❌ **Example:**

```csharp
[HttpGet("{id}")]
public IActionResult GetById(int roleId)  // Parameter name doesn't match route
{
    return Ok($"Role {roleId}");
}
```

**Error Message:**

```
HTTP 404 - Not Found (parameter won't bind correctly)
```

---

### Common Route Conflict Scenarios

| Scenario                                       | Why It Conflicts                      | Solution                                                                    |
| ---------------------------------------------- | ------------------------------------- | --------------------------------------------------------------------------- |
| `[HttpGet("{id}")]` and `[HttpGet("{name}")]`  | Both match any single parameter       | Use different routes: `[HttpGet("id/{id}")]` and `[HttpGet("name/{name}")]` |
| `[HttpGet]` twice                              | Both match base route                 | Add different text: `[HttpGet]` and `[HttpGet("active")]`                   |
| `[HttpGet("{id}")]` and `[HttpGet("details")]` | No conflict - text route has priority | This is OK!                                                                 |

---

### Quick Checklist to Avoid Errors

✅ Every API method has an HTTP verb attribute (`[HttpGet]`, `[HttpPost]`, etc.)

✅ No two methods have the same HTTP verb AND same route pattern

✅ Route parameter names match method parameter names

✅ Different operations use different routes or different HTTP verbs

✅ Use route constraints (`:int`, `:alpha`) to differentiate similar routes

---

## HTTP Status Codes and Response Methods

### Common Status Codes in APIs

| Status Code | Name                  | When to Use                    | .NET Method                                  |
| ----------- | --------------------- | ------------------------------ | -------------------------------------------- |
| **200**     | OK                    | Success - operation completed  | `Ok(data)`                                   |
| **201**     | Created               | Resource created successfully  | `CreatedAtAction(action, routeValues, data)` |
| **204**     | No Content            | Success - no data to return    | `NoContent()`                                |
| **400**     | Bad Request           | Validation failed or bad input | `BadRequest(message)`                        |
| **404**     | Not Found             | Resource not found             | `NotFound(message)`                          |
| **500**     | Internal Server Error | Server-side error              | `StatusCode(500, message)`                   |

### Response Method Examples

| Method                  | Status Code | Example Usage                                            |
| ----------------------- | ----------- | -------------------------------------------------------- |
| `Ok(data)`              | 200         | `return Ok(users);`                                      |
| `Created(uri, data)`    | 201         | `return Created("", new { message = "Created", data });` |
| `NoContent()`           | 204         | `return NoContent();`                                    |
| `BadRequest(msg)`       | 400         | `return BadRequest(new { message = "Email exists" });`   |
| `NotFound(msg)`         | 404         | `return NotFound(new { message = "Not found" });`        |
| `StatusCode(code, msg)` | Custom      | `return StatusCode(500, new { message = "Error" });`     |

**Note:** We use `Created("", data)` as a simple alternative to `CreatedAtAction()` for easier implementation.

---

## Async vs Sync Methods

### Why Use Async?

**Async methods** free up the thread while waiting for database operations, allowing the server to handle more requests.

### Async Method Comparison

| Operation        | Sync Method                  | Async Method                            |
| ---------------- | ---------------------------- | --------------------------------------- |
| Get all          | `_db.Users.ToList()`         | `await _db.Users.ToListAsync()`         |
| Find by ID       | `_db.Users.Find(id)`         | `await _db.Users.FindAsync(id)`         |
| First or default | `_db.Users.FirstOrDefault()` | `await _db.Users.FirstOrDefaultAsync()` |
| Any check        | `_db.Users.Any(x => ...)`    | `await _db.Users.AnyAsync(x => ...)`    |
| Save changes     | `_db.SaveChanges()`          | `await _db.SaveChangesAsync()`          |

### Example: Sync vs Async

**Sync (Not Recommended):**

```csharp
[HttpGet]
public IActionResult GetAll()
{
    var users = _db.Users.ToList(); // Thread waits here
    return Ok(users);
}
```

**Async (Recommended):**

```csharp
[HttpGet]
public async Task<IActionResult> GetAll()
{
    var users = await _db.Users.ToListAsync(); // Thread is freed during DB operation
    return Ok(users);
}
```

### Key Points

✅ Always use `async/await` for database operations in production

✅ Add `async` keyword to method signature

✅ Return `Task<IActionResult>` instead of `IActionResult`

✅ Use `await` keyword before async methods

✅ Add `Async` suffix to EF Core methods (`ToListAsync`, `FindAsync`, etc.)

---

## Task 1: Implement CRUD Endpoints for Roles

Create a complete CRUD API for Roles with exception handling.

### RolesController.cs

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

[ApiController]
[Route("api/[controller]")]
public class RolesController : ControllerBase
{
    private readonly AppDbContext _db;

    public RolesController(AppDbContext db)
    {
        _db = db;
    }

    // GET: api/roles
    [HttpGet]
    public async Task<IActionResult> GetAll()
    {
        try
        {
            var roles = await _db.Roles.ToListAsync();
            return Ok(roles);
        }
        catch (Exception ex)
        {
            return StatusCode(500, new { message = "Error getting roles", error = ex.Message });
        }
    }

    // GET: api/roles/5
    [HttpGet("{id:int}")]
    public async Task<IActionResult> GetById(int id)
    {
        try
        {
            var role = await _db.Roles.FindAsync(id);

            if (role == null)
            {
                return NotFound(new { message = $"Role with ID {id} not found" });
            }

            return Ok(role);
        }
        catch (Exception ex)
        {
            return StatusCode(500, new { message = "Error getting role", error = ex.Message });
        }
    }

    // POST: api/roles
    [HttpPost]
    public async Task<IActionResult> Create([FromBody] Role role)
    {
        try
        {
            // Check ModelState validation
            if (!ModelState.IsValid)
            {
                return BadRequest(ModelState);
            }

            // Check if role name already exists
            var exists = await _db.Roles.AnyAsync(r => r.Name == role.Name);
            if (exists)
            {
                return BadRequest(new { message = "Role name already exists" });
            }

            _db.Roles.Add(role);
            await _db.SaveChangesAsync();

            return Created("", new { message = "Role created successfully", role });
        }
        catch (Exception ex)
        {
            return StatusCode(500, new { message = "Error creating role", error = ex.Message });
        }
    }

    // PUT: api/roles/5
    [HttpPut("{id:int}")]
    public async Task<IActionResult> Update(int id, [FromBody] Role role)
    {
        try
        {
            // Check ModelState validation
            if (!ModelState.IsValid)
            {
                return BadRequest(ModelState);
            }

            if (id != role.Id)
            {
                return BadRequest(new { message = "ID mismatch" });
            }

            var existingRole = await _db.Roles.FindAsync(id);
            if (existingRole == null)
            {
                return NotFound(new { message = $"Role with ID {id} not found" });
            }

            // Check if new name conflicts with another role
            var nameExists = await _db.Roles.AnyAsync(r => r.Name == role.Name && r.Id != id);
            if (nameExists)
            {
                return BadRequest(new { message = "Role name already exists" });
            }

            // Update properties
            existingRole.Name = role.Name;

            await _db.SaveChangesAsync();

            return Ok(new { message = "Role updated successfully", role = existingRole });
        }
        catch (Exception ex)
        {
            return StatusCode(500, new { message = "Error updating role", error = ex.Message });
        }
    }

    // DELETE: api/roles/5
    [HttpDelete("{id:int}")]
    public async Task<IActionResult> Delete(int id)
    {
        try
        {
            var role = await _db.Roles.FindAsync(id);

            if (role == null)
            {
                return NotFound(new { message = $"Role with ID {id} not found" });
            }

            // Check if role has users
            var hasUsers = await _db.Users.AnyAsync(u => u.RoleId == id);
            if (hasUsers)
            {
                return BadRequest(new { message = "Cannot delete role with assigned users" });
            }

            _db.Roles.Remove(role);
            await _db.SaveChangesAsync();

            return Ok(new { message = "Role deleted successfully" });
        }
        catch (Exception ex)
        {
            return StatusCode(500, new { message = "Error deleting role", error = ex.Message });
        }
    }
}
```
## Task 2: Develop User Management APIs

Create APIs for user registration, updating, and listing with validation.

### UsersController.cs

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly AppDbContext _db;

    public UsersController(AppDbContext db)
    {
        _db = db;
    }

    // GET: api/users
    [HttpGet]
    public async Task<IActionResult> GetAll()
    {
        try
        {
            var users = await _db.Users
                .Include(u => u.Role)
                .Select(u => new
                {
                    u.Id,
                    u.Name,
                    u.Email,
                    Role = u.Role != null ? u.Role.Name : null
                })
                .ToListAsync();

            return Ok(users);
        }
        catch (Exception ex)
        {
            return StatusCode(500, new { message = "Error getting users", error = ex.Message });
        }
    }

    // GET: api/users/5
    [HttpGet("{id:int}")]
    public async Task<IActionResult> GetById(int id)
    {
        try
        {
            var user = await _db.Users
                .Include(u => u.Role)
                .Where(u => u.Id == id)
                .Select(u => new
                {
                    u.Id,
                    u.Name,
                    u.Email,
                    Role = u.Role != null ? u.Role.Name : null
                })
                .FirstOrDefaultAsync();

            if (user == null)
            {
                return NotFound(new { message = $"User with ID {id} not found" });
            }

            return Ok(user);
        }
        catch (Exception ex)
        {
            return StatusCode(500, new { message = "Error getting user", error = ex.Message });
        }
    }

    // POST: api/users (Registration)
    [HttpPost]
    public async Task<IActionResult> Register([FromBody] User user)
    {
        try
        {
            // Check ModelState validation
            if (!ModelState.IsValid)
            {
                return BadRequest(ModelState);
            }

            // Check if email already exists
            var emailExists = await _db.Users.AnyAsync(u => u.Email == user.Email);
            if (emailExists)
            {
                return BadRequest(new { message = "Email already registered" });
            }

            // Check if role exists
            var roleExists = await _db.Roles.AnyAsync(r => r.Id == user.RoleId);
            if (!roleExists)
            {
                return BadRequest(new { message = "Invalid Role ID" });
            }

            _db.Users.Add(user);
            await _db.SaveChangesAsync();

            return Created("", new { message = "User registered successfully", user = new
            {
                user.Id,
                user.Name,
                user.Email
            }});
        }
        catch (Exception ex)
        {
            return StatusCode(500, new { message = "Error registering user", error = ex.Message });
        }
    }

    // PUT: api/users/5
    [HttpPut("{id:int}")]
    public async Task<IActionResult> Update(int id, [FromBody] User user)
    {
        try
        {
            // Check ModelState validation
            if (!ModelState.IsValid)
            {
                return BadRequest(ModelState);
            }

            if (id != user.Id)
            {
                return BadRequest(new { message = "ID mismatch" });
            }

            var existingUser = await _db.Users.FindAsync(id);
            if (existingUser == null)
            {
                return NotFound(new { message = $"User with ID {id} not found" });
            }

            // Check if new email conflicts with another user
            var emailExists = await _db.Users.AnyAsync(u => u.Email == user.Email && u.Id != id);
            if (emailExists)
            {
                return BadRequest(new { message = "Email already in use by another user" });
            }

            // Check if role exists
            var roleExists = await _db.Roles.AnyAsync(r => r.Id == user.RoleId);
            if (!roleExists)
            {
                return BadRequest(new { message = "Invalid Role ID" });
            }

            // Update properties
            existingUser.Name = user.Name;
            existingUser.Email = user.Email;
            existingUser.Password = user.Password;
            existingUser.RoleId = user.RoleId;

            await _db.SaveChangesAsync();

            return Ok(new { message = "User updated successfully", user = new
            {
                existingUser.Id,
                existingUser.Name,
                existingUser.Email
            }});
        }
        catch (Exception ex)
        {
            return StatusCode(500, new { message = "Error updating user", error = ex.Message });
        }
    }

    // DELETE: api/users/5
    [HttpDelete("{id:int}")]
    public async Task<IActionResult> Delete(int id)
    {
        try
        {
            var user = await _db.Users.FindAsync(id);

            if (user == null)
            {
                return NotFound(new { message = $"User with ID {id} not found" });
            }

            _db.Users.Remove(user);
            await _db.SaveChangesAsync();

            return Ok(new { message = "User deleted successfully" });
        }
        catch (Exception ex)
        {
            return StatusCode(500, new { message = "Error deleting user", error = ex.Message });
        }
    }
}
```
