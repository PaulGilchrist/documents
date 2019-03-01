# OData Setup for ASP.Net Core

1. Add NuGet package `Microsoft.AspNetCore.OData`
2. In the file `Startup.cs` and function `ConfigureServices()`, add the new line `services.AddOData();` above the line `services.AddMvc` and `services.AddMvcCore` if it exists

3. In the file `Startup.cs` and function `ConfigureServices()`, add the following lines just under the line `services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Latest)`

```cs
.AddJsonOptions(options => {
    options.SerializerSettings.NullValueHandling = NullValueHandling.Ignore;
    options.SerializerSettings.ContractResolver = new Newtonsoft.Json.Serialization.CamelCasePropertyNamesContractResolver();
    options.SerializerSettings.DateTimeZoneHandling = DateTimeZoneHandling.Utc;
    options.SerializerSettings.ReferenceLoopHandling = ReferenceLoopHandling.Ignore;
});
services.AddOData();
```

4. In the file `Startup.cs` function `Configure()` and sub-function `app.UseMvc()`, add the following new lines

```cs
routes => {
routes.Count().Filter().OrderBy().Expand().Select().MaxTop(null);
routes.MapODataServiceRoute("ODataRoute", "odata", modelBuilder.GetEdmModel());
routes.EnableDependencyInjection();
```

5. Adjust `MaxTop` parameter from the above code to meet the API specific requirements
6. In the file `Startup.cs` create a new function for creating the `ODataConventionModelBuilder` using the following example replacing the names and entities appropriately.

```cs
private static IEdmModel GetEdmModels() {
   ODataConventionModelBuilder builder = new ODataConventionModelBuilder();
   builder.Namespace = "ApiTemplate";
   builder.ContainerName = "ApiTemplateContainer";
   builder.EnableLowerCamelCase();
   builder.EntitySet<Address>("addresses");
   builder.EntitySet<User>("users");
   return builder.GetEdmModel();
}
```

7. Ensure any Entities used by OData have their primary key property identified with the `[key]` annotation as in the following example:

```cs
public class User {
   [Key]
   public int Id { get; set; }
   public string FirstName { get; set; }
   public string LastName { get; set; }
   public string Email { get; set; }
   public string Phone { get; set; }
}
```

7a.  Also make sure to add additional model attributes to help with model validation or self-documentation such as in the following example:

```cs
public class User {
    [Key]
    public int Id { get; set; }
    [Display(Name = "First Name")]
    [Required]
    [StringLength(50)]
    public string FirstName { get; set; }
    [Display(Name = "Last Name")]
    [Required]
    [StringLength(50)]
    public string LastName { get; set; }
    [StringLength(150)]
    public string Email { get; set; }
    [StringLength(20)]
     public string Phone { get; set; }
    public List<Address> Addresses { get; set; } = new List<Address>();
}
```

8. Create new controller or change existing controller to extend from `ODataController` rather than `Controller`.  Controller will require a constructor to direct inject the context rather than creating a new context when the controller instance is created.
9. If security is required on the endpoint, it will be annotated using `[Authorize]` and may also optionally contain one or more required roles (ex: `[Authorize(Roles = "Admin,PowerUser")]`


## Additional OData Best Practices

* Make sure any repository objects return iQueriable and not iEnumerable objects back to the controller so OData can manipulate the query before it is executed against the database.  This also ensures the joins are occurring on the database server and not within the API.
* All calls to remote services should be made asynchronously.  A common example of a remote service call would be an Entity Framework call to backend database.  This async call should roll all the way back to the API controller as in the following example:

```cs
public async Task<IActionResult> Get() {
   var users = _db.Users;
   if (!await users.AnyAsync()) {
      return NotFound();
   }
   return Ok(users);
}
```

* Do not hardcode capabilities into the controller, repository, or SQL procedure that can be controlled by OData such as sorting.  An exception to this may be filtering of objects or properties based on security roles.

## Notes

* As of the time of this writing, Swashbuckle/Swagger/Open API did not recognize controllers that extend from `ODataController`, without also adding `Microsoft.AspCore.OData.Versioning.ApiExplorer`, as Swashbuckle used ApiExplorer for reflection to discover each controler and its actions.  There are separate documents discussing the steps for adding versioning to OData and Open API/Swagger.

## References

* Microsoft announcement - [ASP.NET Core OData now Available](https://blogs.msdn.microsoft.com/odatateam/2018/07/03/asp-net-core-odata-now-available/)
* See document [API - Swagger/Open API for ASP.Net Core using Swashbuckle](https://github.com/PaulGilchrist/documents/blob/master/articles/api-swagger-openapi-for-asp-net-core-using-swashbuckle.md) for proper configuration of OData controller function comments and annotation recomendations
* See [GitHub odate-core-template](https://github.com/PaulGilchrist/odata-core-template) for full source code of a working example of all above steps
