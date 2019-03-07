# API - Swagger/Open API for ASP.Net Core using Swashbuckle

As of the time of this writing (3/1/2019), Swashbuckle and OData both fully support ASP.Net Core 2.x and above, but they do not fully support one another.  If adding Swashbuckle to an existing OData API, please first follow the document titled [OData Versioning Setup for ASP.Net Core](https://github.com/PaulGilchrist/documents/blob/master/api-odata-versioning-setup-for-asp-net-core.md).

This document will not only cover a standard installation of Swashbuckle on ASP.Net Core, but all steps needed to ensure it works well with OData

## Swashbuckle Implementation Steps

1. Edit the project’s csproj file adding the following lines to the `<PropertyGroup>` section.  This will enable support for Swagger to generate documentation using any existing function comments like /// <summary>, /// <remarks>, and /// <param>

```xml
<GenerateDocumentationFile>true</GenerateDocumentationFile>
<NoWarn>$(NoWarn);1591</NoWarn>
```

2. Add the following NuGet packages to the project

```cs
Swashbuckle.AspNetCore
Swashbuckle.AspNetCore.Annotations
```

3. In the `Classes` folder (create if needed), add a new class named `ConfigureSwaggerOptions` with the following code (adjusting descriptive content as needed):

```cs
    using Microsoft.AspNetCore.Mvc.ApiExplorer;
    using Microsoft.Extensions.DependencyInjection;
    using Microsoft.Extensions.Options;
    using Swashbuckle.AspNetCore.Swagger;
    using Swashbuckle.AspNetCore.SwaggerGen;

    /// <summary>
    /// Configures the Swagger generation options.
    /// </summary>
    /// <remarks>This allows API versioning to define a Swagger document per API version after the
    /// <see cref="IApiVersionDescriptionProvider"/> service has been resolved from the service container.</remarks>
    public class ConfigureSwaggerOptions : IConfigureOptions<SwaggerGenOptions> {
        readonly IApiVersionDescriptionProvider provider;

        /// <summary>
        /// Initializes a new instance of the <see cref="ConfigureSwaggerOptions"/> class.
        /// </summary>
        /// <param name="provider">The <see cref="IApiVersionDescriptionProvider">provider</see> used to generate Swagger documents.</param>
        public ConfigureSwaggerOptions(IApiVersionDescriptionProvider provider) => this.provider = provider;

        /// <inheritdoc />
        public void Configure(SwaggerGenOptions options) {
            // add a swagger document for each discovered API version
            // note: you might choose to skip or document deprecated API versions differently
            foreach (var description in provider.ApiVersionDescriptions) {
                options.SwaggerDoc(description.GroupName, CreateInfoForApiVersion(description));
            }
        }

        static Info CreateInfoForApiVersion(ApiVersionDescription description) {
            var info = new Info() {
                Title = "OData, Open API, .Net Core demo and training API",
                Version = description.ApiVersion.ToString(),
                Description = "OData, Open API, .Net Core demo and training API",
                Contact = new Contact() { Name = "Paul Gilchrist", Email = "paul.gilchrist@outlook.com" },
                TermsOfService = "Shareware",
                License = new License() { Name = "MIT", Url = "https://opensource.org/licenses/MIT" }
            };
            if (description.IsDeprecated) {
                info.Description += " This API version has been deprecated.";
            }
            return info;
        }
    }
```

4. In the `Classes` folder, add a new class named `SwaggerDefaultValues` with the following code (adjusting descriptive content as needed):

```cs
    using Swashbuckle.AspNetCore.Swagger;
    using Swashbuckle.AspNetCore.SwaggerGen;
    using System.Linq;

    /// <summary>
    /// Represents the Swagger/Swashbuckle operation filter used to document the implicit API version parameter.
    /// </summary>
    /// <remarks>This <see cref="IOperationFilter"/> is only required due to bugs in the <see cref="SwaggerGenerator"/>.
    /// Once they are fixed and published, this class can be removed.</remarks>
    public class SwaggerDefaultValues : IOperationFilter {
        /// <summary>
        /// Applies the filter to the specified operation using the given context.
        /// </summary>
        /// <param name="operation">The operation to apply the filter to.</param>
        /// <param name="context">The current operation filter context.</param>
        public void Apply(Operation operation, OperationFilterContext context) {
            var apiDescription = context.ApiDescription;
            //operation.Deprecated = apiDescription.IsDeprecated();
            if (operation.Parameters == null) {
                return;
            }
            // REF: https://github.com/domaindrivendev/Swashbuckle.AspNetCore/issues/412
            // REF: https://github.com/domaindrivendev/Swashbuckle.AspNetCore/pull/413
            foreach (var parameter in operation.Parameters.OfType<NonBodyParameter>()) {
                var description = apiDescription.ParameterDescriptions.First(p => p.Name == parameter.Name);
                if (parameter.Description == null) {
                    parameter.Description = description.ModelMetadata?.Description;
                }
                if (parameter.Default == null) {
                    parameter.Default = description.DefaultValue;
                }
                parameter.Required |= description.IsRequired;
            }
        }
    }
```

5. In file `Startup.cs` add a new function to the `Startup` classas follows:

```cs
static string XmlCommentsFilePath {
    get {
        var basePath = PlatformServices.Default.Application.ApplicationBasePath;
        var fileName = typeof(Startup).GetTypeInfo().Assembly.GetName().Name + ".xml";
        return Path.Combine(basePath, fileName);
    }
}
```

6. In file `Startup.cs` and function `ConfigureServices`, just under the section `services.AddODataApiExplorer();`, place the following code:

```cs
services.AddTransient<IConfigureOptions<SwaggerGenOptions>, ConfigureSwaggerOptions>();
// Register the Swagger generator, defining 1 or more Swagger documents
services.AddSwaggerGen(options => {
    // add a custom operation filter which sets default values
    options.OperationFilter<SwaggerDefaultValues>();
    options.CustomSchemaIds((x) => x.Name + "_" + Guid.NewGuid());
    // integrate xml comments
    options.IncludeXmlComments(XmlCommentsFilePath);
});
```

7. In file `Startup.cs` and function `Configure()`, just under the section `services.UseMvc();`, place the following code:

```cs
// Enable middleware to serve generated Swagger as a JSON endpoint.
app.UseSwagger();
// Enable middleware to serve swagger-ui (HTML, JS, CSS, etc.), specifying the Swagger JSON endpoint.
app.UseSwaggerUI(options => {
    foreach (var description in provider.ApiVersionDescriptions) {
        options.SwaggerEndpoint(
            $"/swagger/{description.GroupName}/swagger.json",
            description.GroupName.ToUpperInvariant());
    }
    options.DocExpansion(DocExpansion.None);
});
```

9. Ensure each OData controller HTTP action function implements the following comments and attributes appropriate for the given action.  Below are examples of the most common comments and attributes used for each action type.

```cs
/// <summary>Query users</summary>
[HttpGet]
[Route("odata/users")]
[ProducesResponseType(typeof(IEnumerable<User>), 200)] // Ok
[ProducesResponseType(typeof(void), 404)]  // Not Found
[EnableQuery]

/// <summary>Query users by id</summary>
/// <param name="id">The user id</param>
[HttpGet]
[Route("odata/users({id})")]
[ProducesResponseType(typeof(User), 200)] // Ok
[ProducesResponseType(typeof(void), 404)] // Not Found
[EnableQuery(AllowedQueryOptions = AllowedQueryOptions.Select | AllowedQueryOptions.Expand)]

/// <summary>Create a new user</summary>
/// <remarks>
/// Make sure to secure this action before production release
/// </remarks>
/// <param name="user">A full user object</param>
[HttpPost]
[Route("odata/users({id})")]
[ProducesResponseType(typeof(User), 201)] // Created
[ProducesResponseType(typeof(ModelStateDictionary), 400)] // Bad Request
[ProducesResponseType(typeof(void), 401)] // Unauthorized

/// <summary>Edit the user with the given id</summary>
/// <remarks>
/// Make sure to secure this action before production release
/// </remarks>
/// <param name="id">The user id</param>
/// <param name="userDelta">A partial user object.  Only properties supplied will be updated.</param>
[HttpPatch]
[Route("odata/users({id})")]
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
[HttpPut]
[Route("odata/users({id})")]
[ProducesResponseType(typeof(User), 200)] // Ok
[ProducesResponseType(typeof(ModelStateDictionary), 400)] // Bad Request
[ProducesResponseType(typeof(void), 401)] // Unauthorized
[ProducesResponseType(typeof(void), 404)] // Not Found

/// <summary>Delete the given user</summary>
/// <remarks>
/// Make sure to secure this action before production release
/// </remarks>
/// <param name="id">The user id</param>
[HttpDelete]
[Route("odata/users({id})")]
[ProducesResponseType(typeof(void), 204)] // No Content
[ProducesResponseType(typeof(void), 401)] // Unauthorized
[ProducesResponseType(typeof(void), 404)] // Not Found
```

## References

* See document [API - OData Setup for ASP.Net Core](https://github.com/PaulGilchrist/documents/blob/master/articles/api-odata-setup-for-dot-net-core.md) for proper implementation of OData
* See document [OData Versioning Setup for ASP.Net Core](https://github.com/PaulGilchrist/documents/blob/master/api-odata-versioning-setup-for-asp-net-core.md) for proper implementation of OData Versioning

* See [GitHub odate-core-template](https://github.com/PaulGilchrist/odata-core-template) for full source code of a working example of all above steps