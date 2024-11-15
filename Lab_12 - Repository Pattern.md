# Basic LINQ Operators
## Step 1: Prepare folder sturucture
- Create a Infrastructure folder inside root directory.
- Create IRepository and Repository folders inside Infrastructure folder.

## Step 2: Create the Repository Interface
- Here we have created generic interface which contains basic methods.
```csharp
public interface IRepository<T> where T : class
{
    Task<IEnumerable<T>> GetAll();
    Task<T> GetById(int id);
    Task Add(T entity);
    Task Update(T entity);
    Task Delete(int id);
}
```
## Step 3: Implement the Repository
- Let's Implement IRepository<T> interface
```csharp
public class Repository<T> : IRepository<T> where T : class
{
    private readonly ApplicationDbContext _context;
    private readonly DbSet<T> _dbSet;

    public Repository(ApplicationDbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }
    public async Task Add(T entity)
    {
        await _dbSet.AddAsync(entity);
        await _context.SaveChangesAsync();
    }

    public async Task Delete(int id)
    {
        var entity = await GetById(id);
        _dbSet.Remove(entity);
        await _context.SaveChangesAsync();
    }

    public async Task<IEnumerable<T>> GetAll()
    {
        return await _dbSet.ToListAsync();
    }

    public async Task<T> GetById(int id)
    {
        return await _dbSet.FindAsync(id);
    }

    public async Task Update(T entity)
    {
        _dbSet.Update(entity);
        await _context.SaveChangesAsync();
    }
}
```
## Step 4: Implement Specific Repositories

- If we have specific queries or complex logic for certain entities, we can implement specific repositories by inheriting the generic repository.
- Here’s an example for the StudentRepository.

```csharp
public class StudentRepository: Repository<Student>
{
    public StudentRepository(ApplicationDbContext context) : base(context)
    {
    }

    // Add any additional student-specific methods here
    public async Task<IEnumerable<Student>> GetStudentByFilterParams(int? CourseID, int? DepartmentID)
    {
        return await _context.Students.Where(
                x=> (!CourseID.HasValue || x.CourseID == CourseID) &&   // Check CourseID if present
                (!DepartmentID.HasValue || x.DepartmentID == CourseID)  // Check DepartmentID if present
                ).ToListAsync();
    }
}
```

## Step 5: Create Unit of Work
Unit of Work will manage all the repositories at single place as described below:
- First Create IUnitOfWork interface in IRepository folder
```csharp

```

## Step 6: Configure Dependency Injection in Program.cs
- Register both DbContext, Repository, and UnitOfWork in the DI container inside the Program.cs file
  
> AddScoped() registers each repository with a scoped lifetime, ensuring that a new instance of the repository is created for each HTTP request.
```csharp
builder.Services.AddScoped(typeof(IRepository<>), typeof(Repository<>));
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
```

## Step 7: Modify Each controller as required
- Let's modify our Student Controller to implement Repository pattern.
```csharp

```
