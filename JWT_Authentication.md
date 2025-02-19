# How to Implement JWT Token

## Steps to Implementation JWT Token for API project
## Step 1: Add following packages

- Microsoft.AspNetCore.Authentication.JwtBearer
- Microsoft.IdentityModel.Tokens
- System.IdentityModel.Tokens.Jwt

## Step 2: Add Jwt Key, Issuer, Audience in appSetting.json
```csharp
"Jwt": {
  "Key": "ThisIsASecretKeyForJwtTokenGeneration",//replace your key
  "Issuer": "https://localhost:5001",
  "Audience": "https://localhost:5001"
}
```
## Step 3: Add authentication service in Program.cs file
```csharp
// Add authentication services
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]))
        };
    });

//Add Authorization Support
builder.Services.AddAuthorization();

// Add Swagger with JWT authentication support
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo { Title = "My API", Version = "v1" });

    // Configure JWT Authentication in Swagger
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Name = "Authorization",
        Type = SecuritySchemeType.Http,
        Scheme = "Bearer",
        BearerFormat = "JWT",
        In = ParameterLocation.Header,
        Description = "Enter 'Bearer {your JWT token}'"
    });

    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            new string[] {}
        }
    });
});
```
Do write following after var app = builder.Build(); method
```csharp
// Add middleware for Authentication and Authorization
app.UseAuthentication();
app.UseAuthorization();

// Enable Swagger
app.UseSwagger();
app.UseSwaggerUI(options =>
{
    options.SwaggerEndpoint("/swagger/v1/swagger.json", "My API v1");
});
```

## Step 4: Model Class
```csharp
public class UserModel
{
    public string Username { get; set; }
    public string Password { get; set; }
}
```

## Step 5: Add Authorization Controller
```csharp
[Route("api/[controller]")]
[ApiController]
public class AuthController : ControllerBase
{
    private readonly IConfiguration _config;

    public AuthController(IConfiguration config)
    {
        _config = config;
    }

    [HttpPost("login")]
    public IActionResult Login([FromBody] UserModel user)
    {
        if (user.Username == "admin" && user.Password == "password")
        {
            Dictionary<string, Object> result = new Dictionary<string, Object>();
            var token = GenerateJwtToken(user.Username);
            result.Add("token", token);
            //result.Add("user", user); // you can call SP and get required data then add that object in result.
            return Ok(result);
        }

        return Unauthorized();
    }

    private string GenerateJwtToken(string username) // Change this with UserID
    {
        var securityKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_config["Jwt:Key"]));
        var credentials = new SigningCredentials(securityKey, SecurityAlgorithms.HmacSha256);

        var claims = new[]
        {
            new Claim(JwtRegisteredClaimNames.Sub, username),
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
        };

        var token = new JwtSecurityToken(
            issuer: _config["Jwt:Issuer"],
            audience: _config["Jwt:Audience"],
            claims: claims,
            expires: DateTime.UtcNow.AddHours(1),
            signingCredentials: credentials
        );

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```
