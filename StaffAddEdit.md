## Model Change
```csharp
public string? PhotoPath { get;set; }
public IFormFile? Photo { get; set; }
```
## Add Edit Page Change
```
<div class="col-md-4">
    <div class="form-check">
        <label asp-for="Photo" class="form-label">Select Photo</label>
        <input type="file" asp-for="Photo" class="form-control" />
        <label asp-for="PhotoPath" class="form-label" />
    </div>
</div>
```

## List Page Change
```html
<td>
    <td><img src=https://localhost:7155/@staff.PhotoPath  width="100px"/></td>
</td>
```
## Controller
### Add Edit
```csharp
var content = ConvertToMultipartFormData(staff);
```
### Function
```csharp
#region Convert Form to Multipart Form Data to post record as Form Data instead of application/json.
public MultipartFormDataContent ConvertToMultipartFormData(Staff staff)
{
    var formData = new MultipartFormDataContent();

    // Add basic properties to form-data
    formData.Add(new StringContent(staff.StaffID.ToString() ?? ""), "StaffID");
    formData.Add(new StringContent(staff.StaffName ?? ""), "StaffName");
    formData.Add(new StringContent(staff.Email ?? ""), "Email");
    formData.Add(new StringContent(staff.DepartmentID.ToString() ?? ""), "DepartmentID");
    formData.Add(new StringContent(staff.RoleID.ToString() ?? ""), "RoleID");
    formData.Add(new StringContent(staff.MobileNumber ?? ""), "MobileNumber");
    formData.Add(new StringContent(staff.PhotoPath ?? ""), "PhotoPath");
    formData.Add(new StringContent(staff.IsActive.ToString() ?? ""), "IsActive");
    formData.Add(new StringContent(staff.Password ?? ""), "Password");
    formData.Add(new StringContent(staff.Remarks ?? ""), "Remarks");


    // Add uploaded file if available
    if (staff.Photo != null && staff.Photo.Length > 0)
    {
        var streamContent = new StreamContent(staff.Photo.OpenReadStream());
        streamContent.Headers.ContentType = new System.Net.Http.Headers.MediaTypeHeaderValue(staff.Photo.ContentType);
        formData.Add(streamContent, "File", staff.Photo.FileName);
    }

    return formData;
}

#endregion
```

