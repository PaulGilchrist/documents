# Authentication Setup for Bearer Tokens on ASP.Net Core

ASP.NET 2.0 has great support for consuming and validating OAuth 2.0 tokens, thanks to built-in JWT validation middleware.  With OAuth and stateless applications, the client applications is expected to include a `bearer` token in the HTTP `authorization` header of every request.

* See [Angular Azure SSO Authentication and Authorization](https://github.com/PaulGilchrist/documents/blob/master/articles/angular-azure-sso-authentication-and-authorization.md) for more details on client setup.

ASP.NET receives the token and ensures the following:

* It is from an allowed application (audience)
* It has been validated with its issuer
* It has not been tampered with (signature and content hash match)

Within the token are optional claims that can be used within ASP.Net.  Some of the more important ones include the following

* `aud` - Audience of the token. When the token is issued to a client application, the audience is the client_id of the client.
* `iss` - Identifies the token issuer
* `iat` - Issued at time. The time when the JWT was issued. The time is represented as the number of seconds from January 1, 1970 (1970-01-01T0:0:0Z) UTC until the time the token was issued.
* `nbf` - Not before time. The time when the token becomes effective. For the token to be valid, the current date/time must be greater than or equal to the Nbf value. The time is represented as the number of seconds from January 1, 1970 (1970-01-01T0:0:0Z) UTC until the time the token was issued.
* `exp` - Expiration time. The time when the token expires. For the token to be valid, the current date/time must be less than or equal to the exp value. The time is represented as the number of seconds from January 1, 1970 (1970-01-01T0:0:0Z) UTC until the time the token was issued.
* `ver` - Version. The version of the JWT token, typically 1.0.
* `tid` - Tenant identifier (ID) of the Azure AD tenant that issued the token.
* `amr`
* `roles` - roles the user belongs to within the application
* `oid` - Object identifier (ID) of the user object in Azure AD.
* `email` - Email address
* `upn` - User principal name of the user.
* `unique_name` - A unique identifier for that can be displayed to the user. This is usually a user principal name (UPN).
* `sub` - Token subject identifier. This is a persistent and immutable identifier for the user that the token describes. Use this value in caching logic.
* `family_name` - User’s last name or surname. The application can display this value.
* `given_name` - User’s first name. The application can display this value.
* `nonce` - One use security key
* `pwd_exp` - Password expiration date
* `pwd_url` - Password change URL

After receiving the token, any function, class, or the entire application can be secured using the [Authorize] annotation above the function or class.  If no token was received, or the token was invalid, ASP.Net will send a `401 unauthorized` response.  For more granular control, you can add role requirements to the annotation as in the following example:

* `[Authorize(Roles=”Admin,Power User”)]`

If a valid token was received but lacked a required role, the ASP.Net will send a `403 forbidden` response.

To setup OAuth 2.0 JWT token security with ASP.NET 2.0 and above, follow the below steps:

1. In file `appsettings.json` add the following section replacing the example GUIDs with those for your environment

```json
"Security": {
   "AllowedAudiences": "a0122984-1cdd-44f6-865e-e20063c7e38f,e3d0aca6-5501-461e-84a9-c0adfc0e3860",
   "TenantIdentifier": "c96c7ef6-407b-4570-98e4-1221a9de2ec9"
},
```

   * Notice that multiple `AllowedAudiences` are supported when separating them by a `","`

2. Add NuGet package named `Microsoft.AspNetCore.Authentication.JwtBearer`.

3. In file `Startup.cs` function `ConfigureServices` add the following code:

```cs
    // ADAL uses v1.0 and MSAL uses v2.0
    List<string> ValidIssuers = new List<string>();
    ValidIssuers.Add("https://login.microsoftonline.com/" + Configuration.GetValue<string>("Security:TenantIdentifier"));
    ValidIssuers.Add("https://login.microsoftonline.com/" + Configuration.GetValue<string>("Security:TenantIdentifier") + "/v2.0");
    /*
        * IssuerSigningKeys needed for access tokens but not for id tokens
        * Keys are at these two locations, but keys should be the same at both URLs:
        * https://login.microsoftonline.com/mytenant.onmicrosoft.com/discovery/keys
        * https://login.microsoftonline.com/mytenant.onmicrosoft.com/discovery/v2.0/keys
        * Use the "kid" values
    */
    services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
        // Configure OAuth Authentication
        .AddJwtBearer(options => {
            options.Authority = "https://login.microsoftonline.com/" + Configuration.GetValue<string>("Security:TenantIdentifier");
            options.TokenValidationParameters = new TokenValidationParameters {
                ValidAudiences = Configuration.GetValue<string>("Security:AllowedAudiences").Split(','),
                ValidIssuers = ValidIssuers
            };
        });
```

4. In file `Startup.cs` function `Configure()` above `app.UseMvc` add the following code (if not already existing):

```cs
app.UseAuthentication();
```

5. On any API controller endpoint requiring security, add the `[Authorize]` annotation with or without `roles` as required.
6. Add `/// <remarks>` comments above the function noting the required roles when using Swagger/Open API

For even more granular security, if you want direct acces to role membership with a function, it can be accessed as follows:

```cs
var roles = User.Claims.Where(c => c.Type == ClaimsIdentity.DefaultRoleClaimType).FirstOrDefault().Value.Split(',');
```

Similar code can be used to access any of the token's claims.

## References

* See [GitHub odate-core-template](https://github.com/PaulGilchrist/odata-core-template) for full source code of a working example of all above steps
