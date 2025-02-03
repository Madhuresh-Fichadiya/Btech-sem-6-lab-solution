1. Get a list of employee names.
```csharp
var employeeNames = employees.Select(e => e.Name).ToList();
```
2. Get names and their respective departments.
```csharp
var empDetails = employees.Select(e => new { e.Name, e.Department }).ToList();
```
3. Get names of employees earning more than 5000.
```csharp
var highEarners = employees.Where(e => e.Salary > 5000).Select(e => e.Name).ToList();
```
4. Get the names in uppercase.
```csharp
var upperNames = employees.Select(e => e.Name.ToUpper()).ToList();
```
5. Get employee IDs along with salaries as strings.
```csharp
var empSalaries = employees.Select(e => $"ID: {e.EmployeeID}, Salary: {e.Salary}").ToList();
```
6. Get all unique skills.
```csharp
var allSkills = employees.SelectMany(e => e.Skills).Distinct().ToList();
```
7. Get employees who know "C#".
```csharp
var csharpEmployees = employees.Where(e => e.Skills.Contains("C#")).Select(e => e.Name).ToList();
```
8. Get department-wise skill sets.
```csharp
var departmentSkills = employees.GroupBy(e => e.Department)
                                .SelectMany(g => g.SelectMany(e => e.Skills))
                                .Distinct()
                                .ToList();
```
9. Get employees older than 30.
```csharp
var olderEmployees = employees.Where(e => e.Age > 30).ToList();
```
10. Get permanent employees.
```csharp
var permanentEmployees = employees.Where(e => e.IsPermanent).ToList();
```
11. Get employees earning between 4000 and 6000.
```csharp
var midSalaryEmployees = employees.Where(e => e.Salary >= 4000 && e.Salary <= 6000).ToList();
```
12. Get employees whose names start with 'J'.
```csharp
var jEmployees = employees.Where(e => e.Name.StartsWith("J")).ToList();
```
13. Get all integer-type salaries (assuming list contains mixed data).
```csharp
var salaryNumbers = employees.Select(e => (object)e.Salary).OfType<int>().ToList();
```
14. Get all boolean properties.
```csharp
var boolValues = employees.Select(e => (object)e.IsPermanent).OfType<bool>().ToList();
```
15. Get the total salary expense.
```csharp
var totalSalary = employees.Sum(e => e.Salary);
```
16. Find the highest salary.
```csharp
var maxSalary = employees.Max(e => e.Salary);
```
17. Find the lowest salary.
```csharp
var minSalary = employees.Min(e => e.Salary);
```
18. Find the average salary.
```csharp
var avgSalary = employees.Average(e => e.Salary);
```
19. Group employees by department.
```csharp
var deptGroups = employees.GroupBy(e => e.Department);
```
20. Find the count of employees in each department.
```csharp
var deptCount = employees.GroupBy(e => e.Department)
                         .Select(g => new { Department = g.Key, Count = g.Count() })
                         .ToList();
```
21. Get the highest salary in each department.
```csharp
var deptMaxSalary = employees.GroupBy(e => e.Department)
                             .Select(g => new { Department = g.Key, MaxSalary = g.Max(e => e.Salary) })
                             .ToList();
```
22. Get employees sorted by salary in descending order.
```csharp
var sortedEmployees = employees.OrderByDescending(e => e.Salary).ToList();
```
23. Get employees sorted first by department, then by salary.
```csharp
var sortedMulti = employees.OrderBy(e => e.Department).ThenByDescending(e => e.Salary).ToList();
```
24. Find the employee with the highest salary.
```csharp
var topEarner = employees.OrderByDescending(e => e.Salary).FirstOrDefault();
```
25. Find the second-highest salary.
```csharp
var secondHighestSalary = employees.OrderByDescending(e => e.Salary).Skip(1).FirstOrDefault();
```
26. Get all employees with names having at least one vowel.
```csharp
var vowelEmployees = employees.Where(e => "AEIOUaeiou".Any(v => e.Name.Contains(v))).ToList();
```
27. Get department-wise average salary excluding lowest salary.
```csharp
var avgExcludingMin = employees.GroupBy(e => e.Department)
                               .Select(g => new { Department = g.Key, AvgSalary = g.Where(e => e.Salary > g.Min(e => e.Salary)).Average(e => e.Salary) })
                               .ToList();
```
28. Find employees whose total skill length exceeds 10 characters.
```csharp
var longSkillEmployees = employees.Where(e => e.Skills.Sum(s => s.Length) > 10).ToList();
```
29. Get employees whose names are palindromes.
```csharp
var palindromeEmployees = employees.Where(e => e.Name.ToLower() == new string(e.Name.ToLower().Reverse().ToArray())).ToList();
```
30. Get department with the highest number of employees.
```csharp
var topDepartment = employees.GroupBy(e => e.Department)
                             .OrderByDescending(g => g.Count())
                             .Select(g => g.Key)
                             .FirstOrDefault();
```
