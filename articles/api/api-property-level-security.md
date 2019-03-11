# API Property Level Security

Most APIs are secured on the individual controller functions using `[Authorize(Role=””)]` attributes.  This works for the majority of security needs, but sometime it will be required to secure object at a more granular level.  This document will discuss how to secure individual object properties within an HTTP action.  We will use an example here where only an application or user with role of `Admin` is allowed to PATCH a user’s email address, but a `Power User` is allowed to PATCH any of the other user properties.  This is just a single example, but the method described could be applied to any other property level access restriction.

1. In the `Classes` folder, create a new class named `Security` (if it does not pre-exist), and add a function named `HasRole` as follows:

```cs
public static bool HasRole(ClaimsPrincipal user, string role) {
    return ((ClaimsIdentity)user.Identity).HasClaim("http://schemas.microsoft.com/ws/2008/06/identity/claims/role", role);
}
```

2. In the `Models` folder, create a new class named `ForbiddenException` as follows:

```cs
public class ForbiddenException {
    public string SecuredColumn { get; set; }
    public string RoleRequired { get; set; }
    public string Description { get; set; }
}
```

   * This class will be used for sending back more useful information than just a typical `Forbidden (403)` reqponse.  Useful information includes what column or columns were restricted, and what role or roles are needed to apply the requested action.

3. For controller functions that have secured columns, add an additional `ProducesResponseType` attribute to allow Swagger to properly document the object model that would be returned in the event the user or API key was not authorized:

```cs
[ProducesResponseType(typeof(ForbiddenException), 403)]
```

4. When using Bulk update PATCH actions (see [OData Bulk Updates](https://github.com/PaulGilchrist/documents/blob/master/articles/api/api-odata-bulk-updates.md)), we are already looking at each object property being sent in the HTTP request, making it easy to add a security check leveraging the `HasRole` function and `ForbiddenException` class we created above.  Here is an example where we are limiting modification to a User's email address to the `Admin` role only:

```cs
// Example of column level security with appropriate description if forbidden
if ((String.Compare(patchUserProperty.Name, "Email", true)==0) && !Security.HasRole(User, "Admin")) {
    return StatusCode(403, new ForbiddenException { SecuredColumn="email", RoleRequired="Admin", Description="Modification to property 'email' requires role 'Admin'"});
}
```

   * POST and DELETE http actions would most likely not have individual properties secured as by definiiton they requiring creating or deleting the full object.  However, this same method would support the POST action as well.
 
 For bulk updates, securing objects in this way ensured that the whole array of objects are saved or rolled back as a set, but although not recomended, could be committed individually with minor changes to the example PATCH function.  See the document titled "[OData Bulk Updates](https://github.com/PaulGilchrist/documents/blob/master/articles/api/api-odata-bulk-updates.md)" for more details but the full PATCH function example including property leve security is as follows:

```cs
/// <summary>Bulk edit users</summary>
/// <remarks>
/// Make sure to secure this action before production release
/// </remarks>
/// <param name="userList">An object containing an array of partial user objects.  Only properties supplied will be updated.</param>
[HttpPatch]
[ODataRoute("")]
[ProducesResponseType(typeof(IEnumerable<User>), 200)] // Ok
[ProducesResponseType(typeof(ModelStateDictionary), 400)] // Bad Request
[ProducesResponseType(typeof(void), 401)] // Unauthorized - User not authenticated
[ProducesResponseType(typeof(ForbiddenException), 403)] // Forbidden - User does not have required claim roles
[ProducesResponseType(typeof(void), 404)] // Not Found
//[Authorize]
public async Task<IActionResult> Patch([FromBody] UserList userList) {
    // Swagger will document a UserList object model, but what is actually being passed in is a DynamicList since PATCH only passes in the properties that have changed
    //     This means we actually need a DynamicList, so reposition and re-read the body
    //     Full explaination ... https://github.com/PaulGilchrist/documents/blob/master/articles/api/api-odata-bulk-updates.md
    try {
        Request.Body.Position = 0;
        var patchUserList = JsonConvert.DeserializeObject<DynamicList>(new StreamReader(Request.Body).ReadToEnd());
        var patchUsers = patchUserList.value;
        List<User> dbUsers = new List<User>(0);
        System.Reflection.PropertyInfo[] userProperties = typeof(User).GetProperties();
        foreach (JObject patchUser in patchUsers) {
            var dbUser = _db.Users.Find((int)patchUser["id"]);
            if (dbUser == null) {
                return NotFound();
            }
            var patchUseProperties = patchUser.Properties();
            // Loop through the changed properties updating the user object
            foreach (var patchUserProperty in patchUseProperties) {
                // Example of column level security with appropriate description if forbidden
                if ((String.Compare(patchUserProperty.Name, "Email", true)==0) && !Security.HasRole(User, "Admin")) {
                    return StatusCode(403, new ForbiddenException { SecuredColumn="email", RoleRequired="Admin", Description="Modification to property 'email' requires role 'Admin'"});
                }
                foreach (var userProperty in userProperties) {
                    if (String.Compare(patchUserProperty.Name, userProperty.Name, true) == 0) {
                        _db.Entry(dbUser).Property(userProperty.Name).CurrentValue = Convert.ChangeType(patchUserProperty.Value, userProperty.PropertyType);
                        // Could optionally even support deltas within deltas here
                    }
                }
            }
            _db.Entry(dbUser).State = EntityState.Detached;
            _db.Users.Update(dbUser);
            dbUsers.Add(dbUser);
        }
        await _db.SaveChangesAsync();
        return Ok(dbUsers);
    } catch (Exception ex) {
        _telemetryTracker.TrackException(ex);
        return StatusCode(500, ex.Message + "\nSee Application Insights Telemetry for full details");
    }
}
```
