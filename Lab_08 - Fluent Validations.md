Fluent Validation to the `CityModel` in your project.

---

### **Step 1: Install Required NuGet Packages**

You need the following NuGet packages to integrate Fluent Validation with ASP.NET Core:

1. **FluentValidation.AspNetCore**
   ```bash
   Install-Package FluentValidation.AspNetCore
   ```

2. **Newtonsoft.Json** (for JSON serialization and deserialization in your `Save` method)
   ```bash
   Install-Package Microsoft.AspNetCore.Mvc.NewtonsoftJson
   ```

---

### **Step 2: Update `Program.cs`**

Register Fluent Validation services in your `Program.cs` 

#### In `Program.cs`:
```csharp
var builder = WebApplication.CreateBuilder(args);

// Add services to the container
builder.Services.AddControllersWithViews()
    .AddFluentValidation(fv => fv.RegisterValidatorsFromAssemblyContaining<CityModelValidator>())
    .AddNewtonsoftJson(); // Enable Newtonsoft.Json for serialization

builder.Services.Configure<ApiBehaviorOptions>(options =>
{
    options.SuppressModelStateInvalidFilter = true; // Suppress default model validation
});

var app = builder.Build();

// Middleware pipeline
app.UseStaticFiles();
app.UseRouting();
app.MapDefaultControllerRoute();
app.Run();
```

---

### **Step 3: Define the `CityModel`**

Here’s the `CityModel` class:

```csharp
using Microsoft.AspNetCore.Mvc.ModelBinding.Validation;
using System.ComponentModel;

namespace Lab14.City.Models
{
       public class CityModel
   {
       public int? CityID { get; set; }

       [DisplayName("City Name")]
       [ValidateNever]
       public string CityName { get; set; }

       [DisplayName("Country Name")]
       [ValidateNever]
       public int CountryID { get; set; }
       public string? CountryName { get; set; }

       [DisplayName("State Name")]
       [ValidateNever]
       public int StateID { get; set; }
       public string? StateName { get; set; }

       [DisplayName("City Code")]
       [ValidateNever]
       public string CityCode { get; set; }
   }

}
```

---

### **Step 4: Create the Fluent Validator**

Add a validator class for `CityModel`.

```csharp
using FluentValidation;
using Lab14.City.Models;

namespace Lab14.Validators
{
    public class CityModelValidator : AbstractValidator<CityModel>
    {
        public CityModelValidator()
        {
            RuleFor(x => x.CityName)
                .NotEmpty().WithMessage("City Name is required.")
                .MaximumLength(50).WithMessage("City Name cannot exceed 50 characters.");

            RuleFor(x => x.CityCode)
                .NotEmpty().WithMessage("City Code is required.")
                .Length(2, 10).WithMessage("City Code must be between 2 and 10 characters.");

            RuleFor(x => x.CountryID)
                .GreaterThan(0).WithMessage("Please select a valid country.");

            RuleFor(x => x.StateID)
                .GreaterThan(0).WithMessage("Please select a valid state.");
        }
    }
}
```

---

### **Step 5: Create the `CityController`**

Ensure the controller handles cascading dropdowns and form submissions.

```csharp
using Lab14.City.Models;
using Microsoft.AspNetCore.Mvc;
using Newtonsoft.Json;
using System.Text;

namespace Lab14.City.Controllers
{
    public class CityController : Controller
    {
        private readonly HttpClient _httpClient;

        public CityController(HttpClient httpClient)
        {
            _httpClient = httpClient;
        }

        [HttpPost]
        public async Task<IActionResult> Save(CityModel modelCity)
        {
            if (ModelState.IsValid)
            {
                var jsonData = JsonConvert.SerializeObject(modelCity);
                var content = new StringContent(jsonData, Encoding.UTF8, "application/json");

                HttpResponseMessage response;
                if (modelCity.CityID == null)
                {
                    response = await _httpClient.PostAsync("api/city", content);
                }
                else
                {
                    response = await _httpClient.PutAsync($"api/city/{modelCity.CityID}", content);
                }

                if (response.IsSuccessStatusCode)
                {
                    TempData["CityInsertMsg"] = "Record saved successfully.";
                    return RedirectToAction("Index");
                }
                else
                {
                    TempData["ErrorMessage"] = "Failed to save the city.";
                }
            }

            await LoadCountryList();
            return View("CityAddEdit", modelCity);
        }

        private async Task LoadCountryList()
        {
            var countries = await _httpClient.GetStringAsync("api/country");
            ViewBag.CountryList = JsonConvert.DeserializeObject<List<CountryDropDownModel>>(countries);
        }

        [HttpPost]
        public async Task<IActionResult> GetStatesByCountry(int countryId)
        {
            var states = await _httpClient.GetStringAsync($"api/state/{countryId}");
            return Json(JsonConvert.DeserializeObject<List<StateDropDownModel>>(states));
        }
    }
}
```

---

### **Step 6: Update the Razor View**

Add the dropdowns and validation messages.

```html
<form method="post" asp-controller="City" asp-action="Save">
    <div asp-validation-summary="ModelOnly" class="text-danger"></div>
    @Html.HiddenFor(x => x.CityID)

    <div class="form-group">
        <label for="CountryID">Country Name</label>
        <select id="CountryID" name="CountryID" class="form-control" asp-for="CountryID">
            <option value="">Select Country</option>
            @foreach (var country in ViewBag.CountryList)
            {
                <option value="@country.CountryID">@country.CountryName</option>
            }
        </select>
        <span asp-validation-for="CountryID" class="text-danger"></span>
    </div>

    <div class="form-group">
        <label for="StateID">State Name</label>
        <select id="StateID" name="StateID" class="form-control" asp-for="StateID">
            <option value="">Select State</option>
        </select>
        <span asp-validation-for="StateID" class="text-danger"></span>
    </div>

    <div class="form-group">
        <label for="CityName">City Name</label>
        <input type="text" id="CityName" name="CityName" class="form-control" asp-for="CityName" />
        <span asp-validation-for="CityName" class="text-danger"></span>
    </div>

    <div class="form-group">
        <label for="CityCode">City Code</label>
        <input type="text" id="CityCode" name="CityCode" class="form-control" asp-for="CityCode" />
        <span asp-validation-for="CityCode" class="text-danger"></span>
    </div>

    <button type="submit" class="btn btn-success">Save</button>
</form>
```

---

### **Step 7: Add JavaScript for Cascading Dropdown**

```javascript
$(document).ready(function () {
    $('#CountryID').change(function () {
        var countryId = $(this).val();
        if (countryId) {
            $.ajax({
                url: '@Url.Action("GetStatesByCountry", "City")',
                type: 'POST',
                data: { countryId: countryId },
                success: function (data) {
                    $('#StateID').empty().append('<option value="">Select State</option>');
                    $.each(data, function (i, state) {
                        $('#StateID').append('<option value="' + state.stateID + '">' + state.stateName + '</option>');
                    });
                }
            });
        } else {
            $('#StateID').empty().append('<option value="">Select State</option>');
        }
    });
});
```

---

### **Step 8: Test the Application**

1. Run the application.
2. Navigate to the City Add/Edit form.
3. Test form submissions with valid and invalid inputs to confirm FluentValidation works as expected. 
