# Repository Pattern
Basic Repo Idea
(https://procodeguide.com/wp-content/uploads/2021/07/Repository-Pattern-in-ASP.NET-Core-Unit-of-Work-1024x432.png)

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
- First implement the interface for individual entities which contains additional method (other than CRUD) declaration.

```csharp
public interface IStudentRepository:IRepository<Student>
{
    public Task<IEnumerable<Student>> GetStudentByFilterParamsAsync(int? CourseID, int? DepartmentID);
    public bool Exists(int id);
}
```
- Next Implement the Repository with the Interface 
```csharp
public class StudentRepository: Repository<Student>, IStudentRepository
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

    public bool Exists(int id)
    { 
        return _context.Students.Any(x=>x.StudentId.Equals(id));
    }
}
```

## Step 5: Create Unit of Work
Unit of Work will manage all the repositories at single place as described below:
- First Create IUnitOfWork interface in IRepository folder
```csharp
public interface IUnitOfWork
{
    IStudentRepository Students { get; }
    IRepository<Course> Courses { get; } // create entity for course if required and replace with generic repository
    IRepository<Department> Departments { get; } // create entity for department if required and replace with generic repository
    Task<int> Save();
}
```
- Next implment the inteface
```csharp
public class UnitOfWork : IUnitOfWork
{
    protected readonly ApplicationDbContext _context;
    public IStudentRepository Students { get; private set; }
    public IRepository<Course> Courses { get; private set; }
    public IRepository<Department> Departments { get; private set; }

    public UnitOfWork(ApplicationDbContext context)
    {
        _context = context;
        Students = new StudentRepository(_context);
        Courses = new Repository<Course>(_context);
        Departments = new Repository<Department>(_context);
    }
    public async Task<int> Save()
    {
        return await _context.SaveChangesAsync();
    }
}
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
[ApiController]
[Route("api/[controller]/[action]")]
public class StudentController : ControllerBase
{
    // Inject ApplicationDbContext class dependency
    private readonly IUnitOfWork _unitOfWork;
    public StudentController(IUnitOfWork unitOfWork)
    {
        _unitOfWork = unitOfWork;
    }

    // GET: api/Students/GetAll
    [HttpGet]
    public async Task<Object> GetAll()
    {
        return await _unitOfWork.Students.GetAll();
    }

    // GET: api/Students/GetByID/5
    [HttpGet("{id}")]
    public async Task<ActionResult<Student>> GetByID(int id)
    {
        var student = await _unitOfWork.Students.GetById(id);

        if (student == null)
        {
            return NotFound();
        }

        return student;
    }

    // PUT: api/Students/UpdateByPK/5
    [HttpPut("{id}")]
    public async Task<IActionResult> UpdateByPK(int id, Student student)
    {
        if (id != student.StudentId)
        {
            return BadRequest();
        }

        try
        {
            await _unitOfWork.Students.Update(student);
            await _unitOfWork.Save();
        }
        catch (Exception)
        {
            if (!_unitOfWork.Students.Exists(id))
            {
                return NotFound();
            }
            else
            {
                throw;
            }
        }

        return NoContent();
    }

    // POST: api/Students/Add
    [HttpPost]
    public async Task<ActionResult<Student>> Add(Student student)
    {
        await _unitOfWork.Students.Add(student);
        await _unitOfWork.Save();

        return CreatedAtAction("GetByID", new { id = student.StudentId }, student);
    }

    // DELETE: api/Students/DeleteByPK/5
    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteByPK(int id)
    {
        var student = await _unitOfWork.Students.GetById(id);
        if (student == null)
        {
            return NotFound();
        }

        await _unitOfWork.Students.Delete(id);
        await _unitOfWork.Save();

        return NoContent();
    }

}
```
