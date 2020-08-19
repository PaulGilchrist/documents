# API - Swagger/Open API for ASP.Net Core using Swashbuckle

This document will not only cover a standard installation of Swashbuckle on ASP.Net Core 3.1, but all steps needed to ensure it works well with OData

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
namespace API.Classes {
    using Microsoft.AspNetCore.Mvc.ApiExplorer;
    using Microsoft.Extensions.DependencyInjection;
    using Microsoft.Extensions.Options;
    using Microsoft.OpenApi.Models;
    using Swashbuckle.AspNetCore.SwaggerGen;
    using System;

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

        static OpenApiInfo CreateInfoForApiVersion(ApiVersionDescription description) {
            var info = new OpenApiInfo() {
                Title = "OData, Open API, .Net Core demo and training API",
                Version = description.ApiVersion.ToString(),
                Description = "OData, Open API, .Net Core demo and training API",
                Contact = new OpenApiContact() { Name = "Paul Gilchrist", Email = "paul.gilchrist@outlook.com" },
                License = new OpenApiLicense() { Name = "MIT", Url = new Uri("https://opensource.org/licenses/MIT") }
            };
            if (description.ApiVersion.MajorVersion<2 ) {
                info.Description = "This API version has been deprecated.  Please use version V2";
            }
            if (description.ApiVersion.MajorVersion > 2) {
                info.Description = "This API version is in preview.  Please use version V2 for production applications";
            }
            return info;
        }
    }
}
```

4. In the `Classes` folder, add a new class named `SwaggerDefaultValues` with the following code (adjusting descriptive content as needed):

```cs
using Microsoft.AspNetCore.Mvc.ApiExplorer;
using Microsoft.OpenApi.Any;
using Microsoft.OpenApi.Models;
using Swashbuckle.AspNetCore.SwaggerGen;
using System.Linq;

namespace API.Classes {
    /// <summary>
    /// Represents the Swagger/Swashbuckle operation filter used to document the implicit API version parameter.
    /// </summary>
    public class SwaggerDefaultValues : IOperationFilter {
        /// <summary>
        /// Applies the filter to the specified operation using the given context.
        /// </summary>
        /// <param name="operation">The operation to apply the filter to.</param>
        /// <param name="context">The current operation filter context.</param>
        public void Apply(OpenApiOperation operation, OperationFilterContext context) {
            var apiDescription = context.ApiDescription;
            operation.Deprecated |= apiDescription.IsDeprecated();
            if (operation.Parameters == null) {
                return;
            }
            // REF: https://github.com/domaindrivendev/Swashbuckle.AspNetCore/issues/412
            // REF: https://github.com/domaindrivendev/Swashbuckle.AspNetCore/pull/413
            foreach (var parameter in operation.Parameters) {
                var description = apiDescription.ParameterDescriptions.First(p => p.Name == parameter.Name);
                if (parameter.Description == null) {
                    parameter.Description = description.ModelMetadata?.Description;
                }
                if (parameter.Schema.Default == null && description.DefaultValue != null) {
                    parameter.Schema.Default = new OpenApiString(description.DefaultValue.ToString());
                }
                parameter.Required |= description.IsRequired;
            }
        }
    }
}
```

5. In the `Classes` folder, add a new class named `SwaggerIgnoreFilter` with the following code (adjusting descriptive content as needed):

```cs
using Microsoft.OpenApi.Models;
using Newtonsoft.Json;
using Swashbuckle.AspNetCore.SwaggerGen;
using System.Linq;
using System.Reflection;

namespace API.Classes {

    public class SwaggerIgnoreFilter : ISchemaFilter {
        public void Apply(OpenApiSchema schema, SchemaFilterContext context) {
            if (schema?.Properties == null || schema.Properties.Count == 0) {
                return;
            }
            // Hide all models except enums to reduce the browser memory consumption from Swagger UI showing deep nested models
            var excludedList = context.Type.GetProperties(BindingFlags.Public | BindingFlags.Instance)
                .Where(t => t.PropertyType.FullName.Contains("API.Models") && !t.PropertyType.FullName.Contains("Enums"))
                .Select(m => (m.GetCustomAttribute<JsonPropertyAttribute>()?.PropertyName ?? m.Name.ToCamelCase()));
            foreach (var excludedName in excludedList) {
                if (schema.Properties.ContainsKey(excludedName))
                    schema.Properties.Remove(excludedName);
            }
        }
    }

    internal static class StringExtensions {
        internal static string ToCamelCase(this string value) {
            if (string.IsNullOrEmpty(value)) return value;
            return char.ToLowerInvariant(value[0]) + value.Substring(1);
        }
    }
}
```

6. In file `Startup.cs` add a new function to the `Startup` classas follows:

```cs
static string XmlCommentsFilePath {
    get {
        var basePath = PlatformServices.Default.Application.ApplicationBasePath;
        var fileName = typeof(Startup).GetTypeInfo().Assembly.GetName().Name + ".xml";
        return Path.Combine(basePath, fileName);
    }
}
```

7. In file `Startup.cs` and function `ConfigureServices`, just under the section `services.AddODataApiExplorer();`, place the following code:

```cs
services.AddTransient<IConfigureOptions<SwaggerGenOptions>, ConfigureSwaggerOptions>();
// Register the Swagger generator, defining 1 or more Swagger documents
services.AddSwaggerGen(options => {
    // Integrate xml comments (application properties/build tab/output path)
    options.IncludeXmlComments(XmlCommentsFilePath);
    // Add a custom operation filter which sets default values
    options.OperationFilter<SwaggerDefaultValues>();
    // Configure Swagger to filter out $expand objects to improve performance for large highly relational APIs
    options.SchemaFilter<SwaggerIgnoreFilter>();
    // The following two options are only needed is using "Basic" security
    options.AddSecurityDefinition("Basic", new OpenApiSecurityScheme() { In = ParameterLocation.Header, Description = "Please insert Basic token into field", Name = "Authorization", Type = SecuritySchemeType.ApiKey });
    options.AddSecurityRequirement(new OpenApiSecurityRequirement {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Basic"
                }
            },
            Array.Empty<string>()
        }
    });
});
```

8. In file `Startup.cs` and function `Configure()`, just under the section `services.UseMvc();`, place the following code:

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
    options.DefaultModelExpandDepth(2);
    options.DefaultModelsExpandDepth(-1);
    options.DefaultModelRendering(ModelRendering.Model);
    options.DisplayRequestDuration();
    options.DocExpansion(DocExpansion.None);
});
```

9. Ensure each object model implements the following comments appropriate for the given property.  Below is an example object model showing OpenAPI compatible comments.

```cs
    /// <summary>
    /// Represents a user
    /// </summary>
    [Select]
    public class User {
        public User() { }

        /// <summary>
        /// Gets or sets the user identifier
        /// </summary>
        [Key]
        public int Id { get; set; }

        /// <summary>
        /// Gets or sets the user's first name
        /// </summary>
        /// <value>The user's first name.</value>
        [Required]
        [Display(Name = "First Name")]
        [StringLength(50, MinimumLength=2, ErrorMessage="Must be between 2 and 50 characters")]
        public string FirstName { get; set; }

        /// <summary>
        /// Gets or sets the user's middle name or initial
        /// </summary>
        /// <value>The user's middle name or initial.</value>
        [Display(Name = "Middle Name")]
        [StringLength(50, MinimumLength=1, ErrorMessage="Must be between 1 and 50 characters")]
        public string MiddleName { get; set; }

        /// <summary>
        /// Gets or sets the user's last name
        /// </summary>
        /// <value>The user's last name.</value>
        [Required]
        [Display(Name = "Last Name")]
        [StringLength(50, MinimumLength=2, ErrorMessage="Must be between 2 and 50 characters")]
        public string LastName { get; set; }

        /// <summary>
        /// Gets or sets the user's email address
        /// </summary>
        /// <value>The user's email address.</value>
        [StringLength(150, MinimumLength=3)]
        public string Email { get; set; }

        /// <summary>
        /// Gets or sets the user's primary phone number
        /// </summary>
        /// <value>The user's primary phone number.</value>
        [StringLength(20, MinimumLength=7)]
        public string Phone { get; set; }

        /// <summary>
        /// Gets or sets the date and time when the address was first created
        /// </summary>
        /// <value>The address's created date (in UTC)</value>
        public System.DateTime CreatedDate { get; set; }

        /// <summary>
        /// Gets or sets the name of who created the address
        /// </summary>
        /// <value>The address's createdBy name (in UTC)</value>
        [StringLength(50, MinimumLength = 1, ErrorMessage = "CreatedBy must be between 1 and 50 characters")]
        public string CreatedBy { get; set; }

        /// <summary>
        /// Gets or sets the date and time when the address was last modified
        /// </summary>
        /// <value>The address's last modified date (in UTC)</value>
        public System.DateTime LastModifiedDate { get; set; }

        /// <summary>
        /// Gets or sets the name of who last modified the address
        /// </summary>
        /// <value>The address's lastModifiedBy name (in UTC)</value>
        [StringLength(50, MinimumLength = 1, ErrorMessage = "LastModifiedBy must be between 1 and 50 characters")]
        public string LastModifiedBy { get; set; }

        /// <summary>
        /// Gets a list of addresses for this user
        /// </summary>
        /// <value>The <see cref="IList{T}">list</see> of <see cref="Address">addresses</see>.</value>
        [Contained]
        public List<Address> Addresses { get; set; } = new List<Address>();

        /// <summary>
        /// Gets a list of notes for this user
        /// </summary>
        /// <value>The <see cref="IList{T}">list</see> of <see cref="UserNote">notes</see>.</value>
        [Contained]
        public List<UserNote> Notes { get; set; } = new List<UserNote>();

    }
```

10. Ensure each OData controller implements the following comments and attributes appropriate for the given HTTP action/function.  It is especially important to all the `[Produces("application/json")]` attribute, as without it, Swagger will not show the object model being returned.  Below are examples of the most common comments and attributes used for each action type.

```cs
        /// <summary>Query users</summary>
        /// <returns>A list of users</returns>
        /// <response code="200">The users were successfully retrieved</response>
        [HttpGet]
        [ODataRoute("")]
        [Produces("application/json")]
        [ProducesResponseType(typeof(IEnumerable<User>), 200)] // Ok
        [EnableQuery]

        /// <summary>Query users by id</summary>
        /// <param name="id">The user id</param>
        /// <returns>A single user</returns>
        /// <response code="200">The user was successfully retrieved</response>
        /// <response code="404">The user was not found</response>
        [HttpGet]
        [ODataRoute("({id})")]
        [Produces("application/json")]
        [ProducesResponseType(typeof(User), 200)] // Ok
        [ProducesResponseType(typeof(void), 404)] // Not Found
        [EnableQuery]

        /// <summary>Create a new user</summary>
        /// <param name="user">A full user object</param>
        /// <returns>A new user</returns>
        /// <response code="201">The user was successfully created</response>
        /// <response code="400">The user is invalid</response>
        /// <response code="401">Authentication required</response>
        /// <response code="403">Access denied due to inadaquate claim roles</response>
        [HttpPost]
        [ODataRoute("")]
        [Produces("application/json")]
        [ProducesResponseType(typeof(IEnumerable<User>), 201)] // Created
        [ProducesResponseType(typeof(ModelStateDictionary), 400)] // Bad Request
        [ProducesResponseType(typeof(void), 401)] // Unauthorized

        /// <summary>Edit user</summary>
        /// <param name="id">The user id</param>
        /// <param name="userDelta">A partial user object.  Only properties supplied will be updated.</param>
        /// <returns>An updated user</returns>
        /// <response code="200">The user was successfully updated</response>
        /// <response code="400">The user is invalid</response>
        /// <response code="401">Authentication required</response>
        /// <response code="403">Access denied due to inadaquate claim roles</response>
        /// <response code="404">The user was not found</response>
        [HttpPatch]
        [ODataRoute("({id})")]
        [Produces("application/json")]
        [ProducesResponseType(typeof(User), 200)] // Ok
        [ProducesResponseType(typeof(ModelStateDictionary), 400)] // Bad Request
        [ProducesResponseType(typeof(void), 401)] // Unauthorized - User not authenticated
        [ProducesResponseType(typeof(ForbiddenException), 403)] // Forbidden - User does not have required claim roles
        [ProducesResponseType(typeof(void), 404)] // Not Found
        //[Authorize(AuthenticationSchemes = JwtBearerDefaults.AuthenticationScheme + ",BasicAuthentication", Roles = "Admin")]

        /// <summary>Delete the given user</summary>
        /// <param name="id">The user id</param>
        /// <response code="204">The user was successfully deleted</response>
        /// <response code="401">Authentication required</response>
        /// <response code="403">Access denied due to inadaquate claim roles</response>
        /// <response code="404">The user was not found</response>
        [HttpDelete]
        [ODataRoute("({id})")]
        [Produces("application/json")]
        [ProducesResponseType(typeof(void), 204)] // No Content
        [ProducesResponseType(typeof(void), 401)] // Unauthorized
        [ProducesResponseType(typeof(void), 404)] // Not Found
```

11. Ensure in your OData Model Configurations that all `EntitySet` names must match their respective controller names, and these names must be case sensitive or objects passed in the body [FromBody] will show in Swagger as if their properties mustbe passed in the querystring (query).  The name does not have to match the route, allowing routes to still be in camelCase even though the controller and EntitySet names will be in PascalCase.

## References

* See document [API - OData Setup for ASP.Net Core](https://github.com/PaulGilchrist/documents/blob/master/articles/api-odata-setup-for-dot-net-core.md) for proper implementation of OData
* See document [OData Versioning Setup for ASP.Net Core](https://github.com/PaulGilchrist/documents/blob/master/api-odata-versioning-setup-for-asp-net-core.md) for proper implementation of OData Versioning

* See [GitHub odate-core-template](https://github.com/PaulGilchrist/odata-core-template) for full source code of a working example of all above steps