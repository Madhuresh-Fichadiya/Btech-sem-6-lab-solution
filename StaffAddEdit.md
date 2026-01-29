## List Page Change
```html
<td>
    <a asp-action="AddEdit" asp-controller="Staff" asp-route-StaffID="@staff.StaffID" class="btn btn-sm btn-success">Edit</a>
    <a asp-action="Delete" asp-controller="Staff" asp-route-StaffID="@staff.StaffID" class="btn btn-sm btn-danger"
       onclick="return confirm('Are you sure you want to delete this staff member?');">Delete</a>
</td>
```
## Controller
### Add Edit
```csharp
public async Task<IActionResult> AddEdit(int? StaffID)
{
    //Call DDL methods and fill in view bag
    if (StaffID > 0)
    {
        var data = await _httpClient.GetAsync($"Staffs/GetStaff/{StaffID}").Result.Content.ReadAsStringAsync();
        var staff = JsonConvert.DeserializeObject<Models.Staff>(data);
        return View("StaffAddEdit", staff);
    }
    return View("StaffAddEdit");
}
```
### Save
```csharp
[HttpPost]
public async Task<IActionResult> Save(Staff staff)
{
    staff.DepartmentID = 1;
    staff.RoleID = 3;
    if (staff == null)
    {
        return View("StaffAddEdit", staff);
    }
    if (ModelState.IsValid)
    {
        var json = JsonConvert.SerializeObject(staff);
        var content = new StringContent(json, System.Text.Encoding.UTF8, "application/json");
        if (staff.StaffID > 0)
        {
            var response = await _httpClient.PutAsync($"Staffs/PutStaff/{staff.StaffID}", content);
            if (response.IsSuccessStatusCode)
            {
                return RedirectToAction("Index");
            }
            else
            {
                return View("StaffAddEdit", staff);

            }
        }
        else
        {

            var response = await _httpClient.PostAsync("Staffs/PostStaff", content);
            if (response.IsSuccessStatusCode)
            {
                return RedirectToAction("Index");
            }
            else
            {
                return View("StaffAddEdit", staff);
            }
        }
    }
    else
    {
        return View("StaffAddEdit", staff);
    }
}
```
