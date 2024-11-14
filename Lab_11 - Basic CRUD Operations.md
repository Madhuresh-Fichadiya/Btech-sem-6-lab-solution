# Basic CRUD Operations using EF Core
## Step 1: Create a WebAPI project and add following packages
**- Microsoft.EntityFrameworkCore.SqlServer**

**- Microsoft.EntityFrameworkCore.Tools**

**- System.ComponentModel.DataAnnotations**
## Step 2: Prepare Model classes
- Course- CourseId, CourseName
- Student - StudentId, Name, Enrollment, Semester
- StudentCourse - StudentCourseId, StudentId, CourseId, EnrollDate, Grade

```cshrap
[Table("Courses")]
public class Course
{
    #region Properties

    [Key]
    public int CourseId { get; set; }

    [Required]
    [StringLength(100)]
    public string CourseName { get; set; }

    #endregion
}
```
```csharp
[Table("Students")]
public class Student
{
    #region Properties

    [Key]
    public int StudentId { get; set; }

    [Required]
    [StringLength(100)]
    public string Name { get; set; }

    [Required]
    [StringLength(20)]
    public string Enrollment { get; set; }

    [Required]
    public int Semester { get; set; }

    #endregion
}
```
```csharp
[Table("StudentCourses")]
public class StudentCourse
{
    #region Properties

    [Key]
    public int StudentCourseId { get; set; }

    [Required]
    [ForeignKey("Student")]
    public int StudentId { get; set; }

    [Required]
    [ForeignKey("Course")]
    public int CourseId { get; set; }

    [Required]
    public DateTime EnrollDate { get; set; }

    [StringLength(2)]
    public string Grade { get; set; }

    #endregion
}
```
## Step 2: Prepare ApplicationDbContext class
Create a Data folder in root directory and add ApplicationDbContext inside Data folder.
```csharp
public class ApplicationDbContext: DbContext
{
    private readonly IConfiguration configuration;
    public ApplicationDbContext(IConfiguration _configuration)
    {
        configuration = _configuration;
    }

    // DbSet properties represent collections of the entities.
    public DbSet<Course> Courses { get; set; }
    public DbSet<Student> Students { get; set; }
    public DbSet<StudentCourse> StudentCourses { get; set; }

    // Configure the database connection string.
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        // SQL Server connection string.
        optionsBuilder.UseSqlServer(this.configuration.GetConnectionString("Default"));
    }
}

```
## Step 3: Add Connection string in appSettings.json file
```csharp
"ConnectionStrings": {
    "DefaultConnection": "Your_SQL_Server_Connection_String_Here"
}
```
