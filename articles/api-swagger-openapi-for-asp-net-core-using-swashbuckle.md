# API - Swagger/Open API for ASP.Net Core using Swashbuckle

As of the time of this writing (8/9/2018), Swashbuckle and OData both fully support ASP.Net Core 2.x and above, but they do not fully support one another.  This document will not only cover a standard installation of Swashbuckle on ASP.Net Core, but all steps needed to ensure it works well with OData

## Swashbuckle Implementation Steps

1. Edit the project’s csproj file adding the following lines to the `<PropertyGroup>` section.  This will enable support for Swagger to generate documentation using any existing function comments like /// <summary>, /// <remarks>, and /// <param>

```xml
<GenerateDocumentationFile>true</GenerateDocumentationFile>
<NoWarn>$(NoWarn);1591</NoWarn>
```

2. Add the following NuGet packages to the project

```cs
Swashbuckle.AspNetCore
Swashbuckle.AspNetCore.Filters
```

3. If supporting OData, make sure the following code is placed in file `Startup.cs` and function `ConfigureServices`, just under the line `services.AddOData();` and above the line `services.AddMvc()`.

```cs
// Workaround to support OData and Swashbuckle working together: https://github.com/OData/WebApi/issues/1177
services.AddMvcCore(options => {
   foreach (var outputFormatter in options.OutputFormatters.OfType<ODataOutputFormatter>().Where(_ => _.SupportedMediaTypes.Count == 0)) {
      outputFormatter.SupportedMediaTypes.Add(new MediaTypeHeaderValue("application/prs.odatatestxx-odata"));
   }
   foreach (var inputFormatter in options.InputFormatters.OfType<ODataInputFormatter>().Where(_ => _.SupportedMediaTypes.Count == 0)) {
      inputFormatter.SupportedMediaTypes.Add(new MediaTypeHeaderValue("application/prs.odatatestxx-odata"));
   }
}).AddApiExplorer();
```

4. If supporting OData, add a file named `ODataControllerAttribute.cs` to the classes folder, and paste in the following code:

```cs
using System;

namespace OdataCoreTemplate.Classes {
    [AttributeUsage(AttributeTargets.Class)]
    public class ODataControllerAttribute : Attribute {
        // Place this attribute on an OData controller to allow Swagger to document its endpoints
        public Type EntityType { get; set; }

        public ODataControllerAttribute(Type entityType) {
            this.EntityType = entityType;
        }
    }
}
```

5.	If supporting OData, add a file named ` SwaggerODataOperationFilter.cs` to the classes folder, and paste in the following code:

```cs
using Microsoft.AspNet.OData;
using Microsoft.AspNet.OData.Query;
using Swashbuckle.AspNetCore.Swagger;
using Swashbuckle.AspNetCore.SwaggerGen;
using System;
using System.Linq;

namespace OdataCoreTemplate.Classes {
    public class SwaggerODataOperationFilter : IOperationFilter {
        public void Apply(Operation operation, OperationFilterContext context) {
            // Add the standard OData parameters allowed for an controllers when the controller has the [ODataController] attribute and the function has the [EnableQuery] attribute
            var odataControllerAttribute = context.MethodInfo.DeclaringType.GetCustomAttributes(true)
                // .Union(context.MethodInfo.GetCustomAttributes(true))
                .OfType<ODataControllerAttribute>()
                .FirstOrDefault(a => a is ODataControllerAttribute);
            if (odataControllerAttribute != null) {
                if (context.ApiDescription.HttpMethod.Equals("GET", StringComparison.OrdinalIgnoreCase)) {
                    var enableQueryAttribute = context.MethodInfo.GetCustomAttributes(true)
                            .OfType<EnableQueryAttribute>()
                            .FirstOrDefault(a => a is EnableQueryAttribute);
                    if (enableQueryAttribute != null) {
                        if (enableQueryAttribute.AllowedQueryOptions.HasFlag(AllowedQueryOptions.Top))
                            operation.Parameters.Add(new NonBodyParameter() { Name = "$top", Description = "Limits the number of items to be returned", Required = false, Type = "integer", In = "query" });
                        if (enableQueryAttribute.AllowedQueryOptions.HasFlag(AllowedQueryOptions.Skip))
                            operation.Parameters.Add(new NonBodyParameter() { Name = "$skip", Description = "Skips the first /n/ items of the queried collection from the result", Required = false, Type = "integer", In = "query" });
                        if (enableQueryAttribute.AllowedQueryOptions.HasFlag(AllowedQueryOptions.Filter))
                            operation.Parameters.Add(new NonBodyParameter() { Name = "$filter", Description = "Restricts the set of items returned", Required = false, Type = "string", In = "query" });
                        if (enableQueryAttribute.AllowedQueryOptions.HasFlag(AllowedQueryOptions.Select))
                            operation.Parameters.Add(new NonBodyParameter() { Name = "$select", Description = "Restricts the properties returned", Required = false, Type = "string", In = "query" });
                        if (enableQueryAttribute.AllowedQueryOptions.HasFlag(AllowedQueryOptions.OrderBy))
                            operation.Parameters.Add(new NonBodyParameter() { Name = "$orderby", Description = "Sorts the returned items by these properties (asc or desc)", Required = false, Type = "string", In = "query" });
                        if (enableQueryAttribute.AllowedQueryOptions.HasFlag(AllowedQueryOptions.Expand))
                            operation.Parameters.Add(new NonBodyParameter() { Name = "$expand", Description = "Expands navigation properties", Required = false, Type = "string", In = "query" });
                        if (enableQueryAttribute.AllowedQueryOptions.HasFlag(AllowedQueryOptions.Count))
                            operation.Parameters.Add(new NonBodyParameter() { Name = "$count", Description = "Retrieves the total count of items as an attribute in the results.", Required = false, Type = "boolean", In = "query" });
                    }
                }
            }
        }
    }
}
```

4. In file `Startup.cs` and function `ConfigureServices`, add the following code just under the line `services.AddMvc();`:

```cs
// Register the Swagger generator, defining 1 or more Swagger documents
services.AddSwaggerGen(c => {
   c.SwaggerDoc("v1", new Info {
      Title = "OData Core Template",
      Description = "A simple example ASP.NET Core Web API leveraging OData, OAuth, and Swagger/Open API",
      Version = "v1"
   });
   // Set the comments path for the Swagger JSON and UI.
   var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
   var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
   c.IncludeXmlComments(xmlPath);
});
```

5. If supporting OData, in file `Startup.cs`, function `Configure`, add sub-function `services.AddSwaggerGen`, add the following code:

```cs
// Workaround to show OData input parameters in Swashbuckle (waiting on Swashbuckle.AspNetCore.Odata NuGet package)
c.OperationFilter<SwaggerODataOperationFilter>();
```

6. In file `Startup.cs` and function `Configure`, add the following code above the line `app.UseMvc`:

```cs
// Enable middleware to serve generated Swagger as a JSON endpoint.
app.UseSwagger();
// Enable middleware to serve swagger-ui (HTML, JS, CSS, etc.), 
// specifying the Swagger JSON endpoint.
app.UseSwaggerUI(c => {
   c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API V1");
   c.DocExpansion(DocExpansion.None);
});
```

7. Make sure each OData controller is extended from `Controller` and not `ODataController`

8. Make sure each OData controller implements the following class attributes, adjusted appropriatly

```cs
[Route("odata/users")]
[ODataController(typeof(User))]
```

9. Ensure each OData controller HTTP action function implements the following comments and attributes appropriate for the given action.  Below are example of the most common comments and attributes used for each action type.

```cs
/// <summary>Query users</summary>
[HttpGet]
[ProducesResponseType(typeof(IEnumerable<User>), 200)] // Ok
[ProducesResponseType(typeof(void), 404)]  // Not Found
[EnableQuery]

/// <summary>Query users by id</summary>
/// <param name="id">The user id</param>
[HttpGet("{id}")]
[ProducesResponseType(typeof(User), 200)] // Ok
[ProducesResponseType(typeof(void), 404)] // Not Found
[EnableQuery(AllowedQueryOptions = AllowedQueryOptions.Select | AllowedQueryOptions.Expand)]

/// <summary>Create a new user</summary>
/// <remarks>
/// Make sure to secure this action before production release
/// </remarks>
/// <param name="user">A full user object</param>
[HttpPost]
[ProducesResponseType(typeof(User), 201)] // Created
[ProducesResponseType(typeof(ModelStateDictionary), 400)] // Bad Request
[ProducesResponseType(typeof(void), 401)] // Unauthorized

/// <summary>Edit the user with the given id</summary>
/// <remarks>
/// Make sure to secure this action before production release
/// </remarks>
/// <param name="id">The user id</param>
/// <param name="userDelta">A partial user object.  Only properties supplied will be updated.</param>
[HttpPatch("{id}")]
[ProducesResponseType(typeof(User), 200)] // Ok
[ProducesResponseType(typeof(ModelStateDictionary), 400)] // Bad Request
[ProducesResponseType(typeof(void), 401)] // Unauthorized
[ProducesResponseType(typeof(void), 404)] // Not Found

/// <summary>Replace all data for the user with the given id</summary>
/// <remarks>
/// Make sure to secure this action before production release
/// </remarks>
/// <param name="id">The user id</param>
/// <param name="user">A full user object.  Every property will be updated except id.</param>
[HttpPut("{id}")]
[ProducesResponseType(typeof(User), 200)] // Ok
[ProducesResponseType(typeof(ModelStateDictionary), 400)] // Bad Request
[ProducesResponseType(typeof(void), 401)] // Unauthorized
[ProducesResponseType(typeof(void), 404)] // Not Found

/// <summary>Delete the given user</summary>
/// <remarks>
/// Make sure to secure this action before production release
/// </remarks>
/// <param name="id">The user id</param>
[HttpDelete("{id}")]
[ProducesResponseType(typeof(void), 204)] // No Content
[ProducesResponseType(typeof(void), 401)] // Unauthorized
[ProducesResponseType(typeof(void), 404)] // Not Found
```

## References

* See document [API - OData Setup for ASP.Net Core](https://github.com/PaulGilchrist/documents/blob/master/articles/api-odata-setup-for-dot-net-core.md) for proper implementation of OData
* See [GitHub odate-core-template](https://github.com/PaulGilchrist/odata-core-template) for full source code of a working example of all above steps