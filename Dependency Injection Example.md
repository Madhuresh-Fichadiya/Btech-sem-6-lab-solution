## Create a Student.cs class
```csharp
namespace FirstCoreMVCWebApplication.Models
{
    public class Student
    {
        public int StudentId { get; set; }
        public string? Name { get; set; }
        public string? Branch { get; set; }
        public string? Section { get; set; }
        public string? Gender { get; set; }
    }
}
```
## Creating Service Interface:
Create an interface named IStudentRepository.cs within the Models folder. This interface will declare the methods or operations we can perform on the student data.
