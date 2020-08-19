# Authentication Setup for Basic Tokens on ASP.Net Core

.Net Core 3.1 has great support for consuming and validating the HTTP `Authorization` header.  This document will discuss adding support for `Basic` authentication.  Basic authentication is frequently used as API keys to allow API to API communication, where `OAuth` would be more appropriate for User/Browser to server communication to best ensure the ability to protect the key.  In every case, `Basic` authentication should only be passed across an encrypted connection.

After receiving the token, any function, class, or the entire application can be secured by adding the [Authorize] attribute above the function or class.  If no token was received, or the token was invalid, ASP.Net will send a `401 unauthorized` response.  

To setup Basic authenticatiuon with ASP.NET Core 2.0 and above, follow the below steps:

1. If you do not already have a Basic token that will be allowed access, generate one now by first determining an appropriate name and password.
2. You can generate random passwords from [http://passwordsgenerator.net](http://passwordsgenerator.net/)
   * It is recommend to use 10 or more characters including numbers, lowercase, uppercase, and excluding similar and ambiguous characters
3. Base64 bit encode the username and password using the following format `username:password`
   * There are many tools for Base64 encoding and decoding, but one recomendation is [https://www.base64encode.org](https://www.base64encode.org/)
   * Another method is the following powershell script:

```powershell
$Credential = Get-Credential
$EncodedUsernamePassword = [System.Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes($('{0}:{1}' -f $Credential.UserName, $Credential.GetNetworkCredential().Password)))
Write-Output "<!--$EncodedUsernamePassword $($Credential.UserName):$($Credential.GetNetworkCredential().Password)-->"
Write-Output "Authorization: Basic $EncodedUsernamePassword"
```

4. Add the Basic token to the project's `appsettings.json` file within the `Security` section (create if needed) using the property name `AllowedApiKeys`, as in the below example:

```json
  "Security": {
    "AllowedApiKeys": "dGVzdGluZzpwYXNzd29yZA==,c29tZXVzZXI6Z3Vlc3NtZQ=="
  }
```

   * Notice that multiple `AllowedApiKeys` are supported when separating them by a `","`

5. In the `Classes` folder, add a new class named `BasicAuthenticationHandler` with the following code:

```cs
using Microsoft.AspNetCore.Authentication;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
using System;
using System.Net.Http.Headers;
using System.Security.Claims;
using System.Text;
using System.Text.Encodings.Web;
using System.Threading.Tasks;

namespace API.Classes {
    public class BasicAuthenticationHandler : AuthenticationHandler<AuthenticationSchemeOptions> {

        public BasicAuthenticationHandler(
                IConfiguration configuration,
                IOptionsMonitor<AuthenticationSchemeOptions> options,
                ILoggerFactory logger,
                UrlEncoder encoder,
                ISystemClock clock)
                : base(options, logger, encoder, clock) {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        protected override async Task<AuthenticateResult> HandleAuthenticateAsync() {
            if (!Request.Headers.ContainsKey("Authorization")) {
                // Authorization header not in request
                return AuthenticateResult.NoResult();
            }
            if (!AuthenticationHeaderValue.TryParse(Request.Headers["Authorization"], out AuthenticationHeaderValue authHeader)) {
                // Invalid Authorization header
                return AuthenticateResult.NoResult();
            }
            if (!"Basic".Equals(authHeader.Scheme, StringComparison.OrdinalIgnoreCase)) {
                // Not Basic authentication header
                return AuthenticateResult.NoResult();
            }
            var token = authHeader.Parameter;
            string[] AuthroizedApiKeys = Configuration.GetValue<string>("Security:AllowedApiKeys").Split(',');
            //Check passed ApiKey against approved list
            foreach (string AuthroizedApiKey in AuthroizedApiKeys) {
                if (AuthroizedApiKey == token) {
                    try {
                        var credentialBytes = Convert.FromBase64String(token);
                        var credentials = Encoding.UTF8.GetString(credentialBytes).Split(':');
                        var name = credentials[0];
                        var claims = new[] {
                            new Claim(ClaimTypes.Name, name)
                        };
                        var identity = new ClaimsIdentity(claims, Scheme.Name);
                        var principal = new ClaimsPrincipal(identity);
                        return AuthenticateResult.Success(new AuthenticationTicket(principal, Scheme.Name));
                    } catch {
                        return AuthenticateResult.Fail("Forbidden");
                    }
                }
            }
            return AuthenticateResult.Fail("Forbidden");
        }

    }

}
```

   * In future documents, this Basic Authentication class will be enhanced to support role based authorization

6. In file `Startup.cs` function `ConfigureServices` add the following code under OAuth authentication (if exists), or near the beginning of the function:

```cs
// Configure Basic Authentication
services.AddAuthentication("BasicAuthentication")
    .AddScheme<AuthenticationSchemeOptions, BasicAuthenticationHandler>("BasicAuthentication", null);
```

7. In file `Startup.cs` function `Configure` just above `app.UseMvc()` add the following code (if not already existing):

```cs
app.UseAuthentication();
```

8. Applications should always be using encryption, but this becomes critically important when receiving Basic authentication tokens.  To ensure encryption is always used, add the following line in file `Startup.cs` function `Configure` just above `app.UseAuthentication()` if it does not already exist:

```cs
app.UseHttpsRedirection();
```

9. Test by adding an `[Authorize]` attribute to any controller action, and then using a tool like `Postman` to call that API, making sure to add and `Authorization` header to the HTTP request with the value of `Basic dGVzdGluZzpwYXNzd29yZA==`, or any other API key added to `appsettings.json`.

## Advanced Setup - Adding Role Based, Real Time Updatable Authorization

In this section, we will discuss how to add role based authorization to `Basic` authentication.  These roles will be added to the identity in the same manner used by `OAuth` tokens allowing the application to support both authorization methods simultaniously.  We will also discuss how these roles can be cached in memory for best performance, and persisted in SQL for more real time administration of application access rights.

1. In the `Models` folder, create a new class named `ClaimRoles` with using the following code:

```cs
using System.ComponentModel.DataAnnotations;

namespace ODataCoreTemplate.Models {
    public class ClaimRoles {
        [Key]
        [Required]
        [Display(Name = "Name")]
        [StringLength(50)]
        public string Name { get; set; }
        [Display(Name = "Roles")] //Comma delimited string containing 0 or more roles
        [Required]
        [StringLength(50)]
        public string Roles { get; set; }
    }
}
```

2. In the `ApiDbContext` class, add a new `DbSet` for the newly created ClaimRoles model:

```cs
public DbSet<ClaimRoles> ClaimRoles { get; set; }
```

3. In the `ApiDbContext` class, and `OnModelCreating` function, add a new `Entity` for the newly created ClaimRoles model:

```cs
modelBuilder.Entity<ClaimRoles>();
```

4. If using the `MockData` class, add the following code to the bottom of the `AddMockData()` but above `SaveChanges()`.  This is to create at least one account for authentication and authroization testing, and works with the Basic authentication key added to `appsettings.json` earlier in the document.

```cs
// Add a role to the testing user
context.ClaimRoles.Add(new ClaimRoles { Name = "testing", Roles = "Admin,PowerUser" });
```

5. In the `Classes` folder, create a new class named `Securty` using the following code:

```cs
using Microsoft.Extensions.Caching.Memory;
using OdataCoreTemplate.Models;
using System;
using System.Threading.Tasks;

namespace API.Classes {
    public class Security {
        private IMemoryCache _cache;
        private OdataCoreTemplate.Models.ApiDbContext _db;

        public Security(ApiDbContext context, IMemoryCache memoryCache) {
            _cache = memoryCache;
            _db = context;
        }

        public async Task<string[]> GetRoles(string name) {
            // Returns a comma separated list of claim roles as a string
            // First try and get roles from memory cache
            string[] roles = null;
            if (!_cache.TryGetValue("roles-" + name, out roles)) {
                // try and get roles from database
                var claimRoles = await _db.ClaimRoles.FindAsync(name);
                if(claimRoles != null) {
                    roles = claimRoles.Roles.Split(",");
                }
                // Add roles to the cache even if they were not found in the database (roles=null)
                MemoryCacheEntryOptions memoryCacheEntryOptions = new MemoryCacheEntryOptions();
                memoryCacheEntryOptions.SetSlidingExpiration(new TimeSpan(4, 0, 0));
                _cache.Set("roles-" + name, roles, memoryCacheEntryOptions);
            }
            return roles;
        }
    }
}
```

6. In the `Startus.cs` file, `ConfigureServices()` function just below `AddDbContext`, add the following lines of code adding support for in memory caching and the newly created `Security` class:

```cs
services.AddMemoryCache();
services.AddSingleton<Security>();
```

7. Modify the `BasicAuthenticationHandler` class to direct inject the constructor of the newly created `Security` class, saving it to a private property named `_security`.
8. Modify the function named `HandleAuthenticateAsync()`, adding the follwing code after `identity` is set and before `principal` is set.

```cs
var roles = await _security.GetRoles(name);
if (roles == null) {
    // Null means the identity was not found
    return AuthenticateResult.Fail("Forbidden");
}
// Blank string means the identity was found but has no special roles
foreach (var role in roles) {
    identity.AddClaim(new Claim("http://schemas.microsoft.com/ws/2008/06/identity/claims/role", role));
}
```

9. Test by adding an `[Authorize(Role="Admin")]` attribute to any controller action, and then using a tool like `Postman` to call that API, making sure to add an `Authorization` header to the HTTP request with the value of `Basic dGVzdGluZzpwYXNzd29yZA==`, or any other API key added to both `appsettings.json` with roles in the SQL database.