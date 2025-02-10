# Repository Pattern with Asynchronous Methods in ASP.NET Core Web API

## Flow of Web API Using Repository Pattern
![image](https://github.com/user-attachments/assets/733b6b29-556a-4919-bdd1-68eb992b3542)

### Agenda

- What is Repository Pattern?
- Why should we use it?
- Creating Web API Project.
- Prerequisites

## What is Repository Pattern?

The repository pattern is a software design pattern that acts as an abstraction layer between your data access layer and the business logic layer in an ASP.NET Core Web API. It hides the details of how exactly the data is saved or retrieved from the underlying data source. The details of how the data is stored and retrieved are in the respective repository. This means your business logic doesn’t care whether it’s talking to SQL Server, Oracle, or even a mock object for testing purposes.

## Why should we use it?

By using the repository pattern, we are promoting a more loosely coupled approach to access our data from the database. This leads to more clean code and makes it easier to test.

### Benefits

- **Loose Coupling**: By separating data access concerns, your code becomes more loosely coupled, which makes it easier to maintain, test, and modify in the future.
- **Reusability**: You can create generic repository interfaces and concrete repository classes that implement them. This allows you to reuse the same basic functionality for different data entities.
- **Testability**: Since the repository pattern abstracts the data access layer, you can easily create mock repositories for unit testing your business logic without needing a real database connection.

## Creating Web API Project

### Setting Up the Project

1. Create ASP.NET Core Web API Project

### Installing Dependencies

Install the following dependencies via NuGet Package Manager:

- Microsoft.EntityFrameworkCore
- Microsoft.EntityFrameworkCore.SqlServer
- Microsoft.EntityFrameworkCore.Design
- Microsoft.EntityFrameworkCore.Tools

### Defining Entity Models

Create models to represent your database tables. Follow these steps:

1. Right-click on the solution and choose "Add" > "New Folder".
2. Name the folder as "Models".
3. Inside this folder, create files for each entity (e.g., `Employee.cs`, `Department.cs`).

**Example Entity Models**:

```csharp
public class Employee
{
    public int EmployeeId { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
    public DateTime DateOfBrith { get; set; }
    public Gender Gender { get; set; }
    public int DepartmentId { get; set; }
    public string PhotoPath { get; set; }
}

public class Department
{
    public int DepartmentId { get; set; }
    public string DepartmentName { get; set; }
}
```

### Adding DbContext

Create a folder named "Data" and add `AppDbContext.cs`.

**AppDbContext.cs**:

```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options)
    {
    }

    public DbSet<Employee> Employees { get; set; }
    public DbSet<Department> Departments { get; set; }
}
```

In your `appsettings.json` file, add the connection string:

```json
"ConnectionStrings": {
    "DBConnection": "Server=//Your SQL Server name;Initial Catalog=//Your Db Table;Trusted_Connection=True;Encrypt=False"
}
```

In `Program.cs`, configure the DbContext:

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(Configuration.GetConnectionString("DBConnection")));
```

### Create and Execute Database Migrations

Use these commands to create and execute migrations:

```bash
Add-Migration InitialCreate
Update-Database
```

## Implementing Repository Pattern

### Create Repository Interface

Create a folder named "Repository" and then create a subfolder "Interface" within it. Add interfaces `IEmployeeRepository.cs` and `IDepartmentRepository.cs`.

**Example Interface**:

```csharp
public interface IEmployeeRepository
{
    Task<IEnumerable<Employee>> GetEmployees();
    Task<Employee> GetEmployee(int employeeId);
    Task<Employee> AddEmployee(Employee employee);
    Task<Employee> UpdateEmployee(Employee employee);
    void DeleteEmployee(int employeeId);
}

public interface IDepartmentRepository
{
    IEnumerable<Department> GetDepartments();
    Department GetDepartment(int departmentId);
}
```

### Create Repository Class

Inside the "Repository" folder, create a subfolder "Services". Add classes `EmployeeRepository.cs` and `DepartmentRepository.cs`.

**Example Implementation for EmployeeRepository**:

```csharp
public class EmployeeRepository : IEmployeeRepository
{
    private readonly AppDbContext appDbContext;

    public EmployeeRepository(AppDbContext appDbContext)
    {
        this.appDbContext = appDbContext;
    }

    public async Task<IEnumerable<Employee>> GetEmployees()
    {
        return await appDbContext.Employees.ToListAsync();
    }

    public async Task<Employee> GetEmployee(int employeeId)
    {
        return await appDbContext.Employees
            .FirstOrDefaultAsync(e => e.EmployeeId == employeeId);
    }

    public async Task<Employee> AddEmployee(Employee employee)
    {
        var result = await appDbContext.Employees.AddAsync(employee);
        await appDbContext.SaveChangesAsync();
        return result.Entity;
    }

    public async Task<Employee> UpdateEmployee(Employee employee)
    {
        var result = await appDbContext.Employees
            .FirstOrDefaultAsync(e => e.EmployeeId == employee.EmployeeId);

        if (result != null)
        {
            result.FirstName = employee.FirstName;
            result.LastName = employee.LastName;
            result.Email = employee.Email;
            result.DateOfBrith = employee.DateOfBrith;
            result.Gender = employee.Gender;
            result.DepartmentId = employee.DepartmentId;
            result.PhotoPath = employee.PhotoPath;

            await appDbContext.SaveChangesAsync();

            return result;
        }

        return null;
    }

    public async void DeleteEmployee(int employeeId)
    {
        var result = await appDbContext.Employees
            .FirstOrDefaultAsync(e => e.EmployeeId == employeeId);
        if (result != null)
        {
            appDbContext.Employees.Remove(result);
            await appDbContext.SaveChangesAsync();
        }
    }
}
```

### Configure Services in `Program.cs`

Add the following lines to configure the repository services:

```csharp
builder.Services.AddScoped<IDepartmentRepository, DepartmentRepository>();
builder.Services.AddScoped<IEmployeeRepository, EmployeeRepository>();
```

## Creating API Controllers

### EmployeesController

In the "Controllers" folder, create `EmployeesController.cs`.

**Example Controller for Employees**:

```csharp
[Route("api/[controller]")]
[ApiController]
public class EmployeesController : ControllerBase
{
    private readonly IEmployeeRepository employeeRepository;

    public EmployeesController(IEmployeeRepository employeeRepository)
    {
        this.employeeRepository = employeeRepository;
    }

    [HttpGet]
    public async Task<ActionResult> GetEmployees()
    {
        try
        {
            return Ok(await employeeRepository.GetEmployees());
        }
        catch (Exception)
        {
            return StatusCode(StatusCodes.Status500InternalServerError, 
                "Error retrieving data from the database");
        }
    }

    [HttpGet("{id:int}")]
    public async Task<ActionResult<Employee>> GetEmployee(int id)
    {
        try
        {
            var result = await employeeRepository.GetEmployee(id);
            if (result == null) return NotFound();
            return result;
        }
        catch (Exception)
        {
            return StatusCode(StatusCodes.Status500InternalServerError,
                "Error retrieving data from the database");
        }
    }

    [HttpPost]
    public async Task<ActionResult<Employee>> CreateEmployee(Employee employee)
    {
        try
        {
            if (employee == null) return BadRequest();
            var createdEmployee = await employeeRepository.AddEmployee(employee);
            return CreatedAtAction(nameof(GetEmployee), new { id = createdEmployee.EmployeeId }, createdEmployee);
        }
        catch (Exception)
        {
            return StatusCode(StatusCodes.Status500InternalServerError,
                "Error creating new employee record");
        }
    }

    [HttpPut("{id:int}")]
    public async Task<ActionResult<Employee>> UpdateEmployee(int id, Employee employee)
    {
        try
        {
            if (id != employee.EmployeeId) return BadRequest("Employee ID mismatch");
            var employeeToUpdate = await employeeRepository.GetEmployee(id);
            if (employeeToUpdate == null) return NotFound($"Employee with Id = {id} not found");
            return await employeeRepository.UpdateEmployee(employee);
        }
        catch (Exception)
        {
            return StatusCode(StatusCodes.Status500InternalServerError, "Error updating data");
        }
    }

    [HttpDelete("{id:int}")]
    public async Task<ActionResult<Employee>> DeleteEmployee(int id)
    {
        try
        {
            var employeeToDelete = await employeeRepository.GetEmployee(id);
            if (employeeToDelete == null)
                return NotFound($"Employee with Id = {id} not found");
            return await employeeRepository.DeleteEmployee(id);
        }
        catch (Exception)
        {
            return StatusCode(StatusCodes.Status500InternalServerError, "Error deleting data");
        }
    }
}
```

### DepartmentsController

In the "Controllers" folder, create `DepartmentsController.cs`.

**Example Controller for Departments**:

```csharp
[Route("api/[controller]")]
[ApiController]
public class DepartmentsController : ControllerBase
{
    private readonly IDepartmentRepository departmentRepository;

    public DepartmentsController(IDepartmentRepository departmentRepository)
    {
        this.departmentRepository = departmentRepository;
    }

    [HttpGet]
    public async Task<ActionResult> GetDepartments()
    {
        try
        {
            return Ok(await departmentRepository.GetDepartments());
        }
        catch (Exception)
        {
            return StatusCode(StatusCodes.Status500InternalServerError, 
                "Error retrieving data from the database");
        }
    }

    [HttpGet("{id:int}")]
    public async Task<ActionResult<Department>> GetDepartment(int id)
    {
        try
        {
            var result = await departmentRepository.GetDepartment(id);
            if (result == null) return NotFound();
            return result;
        }
        catch (Exception)
        {
            return StatusCode(StatusCodes.Status500InternalServerError,
                "Error retrieving data from the database");
        }
    }
}
```

## Conclusion

**Repository Pattern Flow**: 

- **Controller** → **Repository** → **EF Core** → **SQL Server**

The repository pattern is an essential design pattern for scalable and maintainable applications, particularly for complex data access scenarios.
