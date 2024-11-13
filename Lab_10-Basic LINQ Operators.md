# Basic LINQ Operators
## LINQ Queries on Projection operators

```sql
// 1. Display FirstName of all employees.
var q1 = context.Employee.Select(x => x.FirstName);
foreach (var employee in q1)
{
Console.WriteLine("\n {0}", employee);
}
// 2. Select ActNo, FirstName and Salary of all employees to a new class and
display it.
var q2 = context.Employee.Select(x => new Employee { AccountNo = x.AccountNo,
FirstName = x.FirstName, Salary = x.Salary });
foreach (var employee in q2)
{
Console.WriteLine("AccountNo: {0}, First Name: {1}, Salary: {2}",
employee.AccountNo, employee.FirstName, employee.Salary);
}
// 3. Display data in following format => {Anil} works in {Admin} Department.
var q3 = context.Employee.Select(x => new { ans = x.FirstName + " works in " +
x.Department + " Deapartmet" });
foreach (var item in q3)
{
Console.WriteLine(item.ans);
}
// 4. Select Employee Full Name, Email and Department as anonymous and display
it.
var q4 = context.Employee.Select(x => "Name: " + x.FirstName + " " + x.LastName +
"\tEmail: " + x.Email + "\tDept: " + x.Department).ToList();
foreach (var item in q4)
Console.WriteLine(item);
// 5. Display employees with their joining date.
var q5 = context.Employee.Select(x => x.FirstName + " " + x.JoiningDate);
foreach (var item in q5)
Console.WriteLine(item);
```
