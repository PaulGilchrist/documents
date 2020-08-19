# Versioning Setup for OData on ASP.Net Core

At some point in time every API will go through its first breaking change and require at least temporarily support for two or more versions allowing time for accessing applications to update their API requests.  When done properly, an API should be able to add objects or properties without it breaking any requesting applications.  Only when an object or property is being removed, or changed would a new API version be required.

Any requestor of the API will need a method for passing the requested version as part of the request.  This is usually done in either the HTTP header, querystring, or URL.  Swagger/Open API requires the version to be passed in the URL or querystring because there is no way to expose different object contracts on the exact same HTTP request URL.  The Swagger/Open API UI and document are always specific to a single version with that version identified in the URL.  Example:

```http
https://<api-root-url>/swagger/docs/v1
```

## Recomendations

* The version should be included in the QueryString of the request URL as implemented by Microsoft's OData versioning library.  Example: `https://<api-site-url>//odata/users?api-version=1.0`
* Increment version number in response to any breaking API change
* Major.Minor version number.  Major number means previous version is deprecated
* Open API/Swagger should document every active API version, making it easy for a developer to discover the latest version, and implementation changes between versions.
* Changes to the contract of an API are considered a breaking change. Changes that impact the backwards compatibility of an API are a breaking change.
   * Breaking changes
      * Removing or renaming APIs or API parameters
      * Changes in behavior for an existing API
      * Changes in Error Codes and Fault Contracts
      * Anything that would violate the Principle of Least Astonishment
   * Non-breaking changes
      * Adding new endpoints
      * Adding properties to existing endpoints
   * Clients should be prepared for services to make incremental changes to their model. In particular, clients should be prepared to receive properties and derived types not previously defined by the service.
   * Services should not change their data model depending on the authenticated user.

## Setup Steps

1. Add NuGet package `Microsoft.AspNetCore.OData.Versioning.ApiExplorer`
2. In the file `Startup.cs` and function `ConfigureServices()`, add the line `EnableEndpointRouting = false` to `services.AddMvc()` as in the following example:

```cs
services.AddMvc(options => options.EnableEndpointRouting = false)
```

3. Add the following code after `services.AddMvc()` and before `services.AddOData()`:

```cs
    services.AddApiVersioning(options => {
        //options.ApiVersionReader = new QueryStringApiVersionReader();
        options.ReportApiVersions = true;
        // required when adding versioning to and existing API to allow existing non-versioned queries to succeed (not error with no version specified)
        options.AssumeDefaultVersionWhenUnspecified = true;
        options.DefaultApiVersion = ApiVersions.V1;
        // Needed to fix issue #1754 - https://github.com/OData/WebApi/issues/1754
        options.RegisterMiddleware = false;
    });
```

4. Append `.EnableApiVersioning();` to `services.AddOData()` and immediately following it add the below additional code:

```cs
services.AddODataApiExplorer(options => {
    // add the versioned api explorer, which also adds IApiVersionDescriptionProvider service
    // note: the specified format code will format the version as "'v'major[.minor][-status]"
    options.GroupNameFormat = "'v'VVV";
});
```

5. Add a new `Configuration` folder to the project
6. Add a new class named `ApiVersions` in the new `Configuration` folder.  As new API versions are added, they will be added to this file, but for now start the class looking like the following:

```cs
static class ApiVersions {
    internal static readonly ApiVersion V1 = new ApiVersion(1, 0);
}
```

7. Add a new class named `ODataModelConfigurations` in the new `Configuration` folder.  Configuration within this class will apply globally for OData, as well as applying to all versions unless using the passed in `ApiVersion` parameter in a conditional.  Start the class following the below code replacing the entity sets with what your current entity sets:

```cs
    /// <summary>
    /// Represents the model configuration for all configurations.
    /// </summary>
    public class ODataModelConfigurations : IModelConfiguration {
        /// <summary>
        /// Applies model configurations using the provided builder for the specified API version.
        /// </summary>
        /// <param name="builder">The <see cref="ODataModelBuilder">builder</see> used to apply configurations.</param>
        /// <param name="apiVersion">The <see cref="ApiVersion">API version</see> associated with the <paramref name="builder"/>.</param>
        public void Apply(ODataModelBuilder builder, ApiVersion apiVersion) {
            builder.Namespace = "ApiTemplate";
            builder.ContainerName = "ApiTemplateContainer";
            // Called once for each apiVersion, so this is the best place to define the EntiitySet differences from version to version
            // IMPORTANT - The entity set name is case sensitive and must match the controller name or the POST properties will not show as coming [FromBody] but rather from the (query) in Swagger
            var user = builder.EntitySet<User>("Users").EntityType;
            user.Count().Expand(5).Filter().OrderBy().Page(null,100).Select();

            var address = builder.EntitySet<Address>("Addresses").EntityType;
            address.Count().Expand(5).Filter().OrderBy().Page(null, 100).Select();

            //Example of how we can remove a field in the data model that may still exist in the database, supporting zero downtime deployments
            //     Adding a property would not be considered a breaking change and not warrant a new ApiVersion
            // if (apiVersion>=ApiVersions.V2) {
            //     user.Ignore(o => o.MiddleName);
            //     address.Ignore(o => o.StreetName2);
            // }
        }
    }
```

8. In the file `Startup.cs`, function `Configure()`, and section `app.UseMvc()` change `MapODataServiceRoute` to `MapVersionedODataRoutes` and change `GetEdmModel` to `GetEdmModels`.  This will allow for parsing of the `ApiVersions` class and all `IModelConfiguration` classes created in the previous steps.

9. Create a folder named `V1` off the `Controllers` folder.  This folder can be named differently, but should reflect the first version of the API.
10. Move all current controllers into the new `V1` folder and change their namesspaces accordingly.
11. Add the attribute `[ApiVersion("1.0")]` to every controller in the `V1` folder.  

In the future, a new folder (`V2` for example) would be added under the `Controllers` folder reflecting the new version. As controllers are added to the new folder, they would have attributes of `[ApiVersion("2.0")]` and the matching controller in the `V1` folder would have its attribute changed to `[ApiVersion("1.0", Deprecated = true)]`.  Differences in the controller would be handled either within the controller actions themselves, or in the `IModelConfiguration` class specific to that controller.  This is demonstrated in the example code above where a user's middle name is hidden from the `V2` controller, but visible in the `V1` version of the same controller.  Similarly in the controller class itself, different actions could exist, different security requirements, or even completely different implementations.

Controllers do not have to exist in every version, allowing for deprecated controllers to over time be removed, and new controllers to be implemented

## References
* [Microsoft API Guidelines](https://github.com/Microsoft/api-guidelines)
* [Microsoft API Versioning Guidelines](https://github.com/Microsoft/api-guidelines/blob/vNext/Guidelines.md#12-versioning)
* [Versioning an API With ASP.Net and swagger](https://medium.com/@MAliBazzi/versioning-an-api-with-asp-net-and-swagger-the-unusual-way-613e34e2d61a)
* [API Versioning in ASP.Net Core with Nice Swagg](https://blog.jimismith.me/blogs/api-versioning-in-aspnet-core-with-nice-swagg)