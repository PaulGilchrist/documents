# Authentication Setup for Basic Tokens on ASP.Net Core

ASP.NET 2.x has great support for consuming and validating the HTTP `Authorization` header.  This document will discuss adding support for `Basic` authentication.  Basic authentication is frequently used as API keys to allow API to API communication, where `OAuth` would be more appropriate for User/Browser to server communication to best ensure the ability to protect the key.  In every case, `Basic` authentication should only be passed across an encrypted connection.

After receiving the token, any function, class, or the entire application can be secured by adding the [Authorize] attribute above the function or class.  If no token was received, or the token was invalid, ASP.Net will send a `401 unauthorized` response.  

To setup Basic authenticatiuon with ASP.NET Core 2.0 and above, follow the below steps:

1. If you do not already have a Basic token that will be allowed access, generate one now by first determining an appropriate name and password.
2. You can generate random passwords from [http://passwordsgenerator.net](http://passwordsgenerator.net/)
   * It is recommend to use 10 or more characters including numbers, lowercase, uppercase, and excluding similar and ambiguous characters
3. Base64 bit encode the username and password using the following format `username:password`
   * There are many tools for Base64 encoding and decoding, but one recomendation is [https://www.base64encode.org](https://www.base64encode.org/)
   * Another method is the following powershell script:

```ps
$Credential = Get-Credential
$EncodedUsernamePassword = [System.Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes($('{0}:{1}' -f $Credential.UserName, $Credential.GetNetworkCredential().Password)))
Write-Output "<!--$EncodedUsernamePassword $($Credential.UserName):$($Credential.GetNetworkCredential().Password)-->"
Write-Output "Authorization: Basic $EncodedUsernamePassword"
```

4. Add the Bsic token to the project's `appsettings.json` file within the `Security` section (create if needed) using the property name `AllowedApiKeys`, as in the below example:

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
                        var username = credentials[0];
                        var claims = new[] {
                            new Claim(ClaimTypes.Name, username)
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

7. Applications should always be using encryption, but this becomes critically important when receiving Basic authentication tokens.  To ensure encryption is always used, add the following line in file `Startup.cs` function `Configure` just above `app.UseAuthentication()` if it does not already exist:

```cs
app.UseHttpsRedirection();
```