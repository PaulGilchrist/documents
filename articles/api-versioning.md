# API Versioning

At some point in time every API will go through its first breaking change and require at least temporarily support for two or more versions allowing time for accessing applications to update their API requests.  When done properly, and API should be able to add objects or properties without it breaking any requesting applications.  Only when an object or property is being removed, or changed would a new API version be required.

Any requestor of the API will need a method for passing the requested version as part of the request.  This is usually done in either the HTTP header, querystring, or URL.  Swagger/Open API requires the version to be passed in the URL because there is no way to expose different object contracts on the same HTTP action.  The Swagger/Open API UI and document are always specific to a single version with that version identified in the URL.  Example:

```http
https://<api-root-url>/swagger/docs/v1
```

## Recomendations

* The version should be embedded in the path of the request URL, at the end of the service root: `https://<api-root-url>/v1.0/products/users`
* Increment version number in response to any breaking API change
* Major.Minor version number.  Major number means previous version is deprecated
* Online documentation of versioned services should indicate the current support status of each previous API version and provide a path to the latest version.
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

## References
* [Microsoft API Guidelines](https://github.com/Microsoft/api-guidelines)
* [Microsoft API Versioning Guidelines](https://github.com/Microsoft/api-guidelines/blob/vNext/Guidelines.md#12-versioning)
* [Versioning an API With ASP.Net and swagger](https://medium.com/@MAliBazzi/versioning-an-api-with-asp-net-and-swagger-the-unusual-way-613e34e2d61a)
* [API Versioning in ASP.Net Core with Nice Swagg](https://blog.jimismith.me/blogs/api-versioning-in-aspnet-core-with-nice-swagg)
