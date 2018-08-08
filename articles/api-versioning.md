# Draft - API Versioning

At some point in time every API will go through its first breaking change and require at least temporarily support for two or more versions allowing time for accessing applications to update their API requests.  When done properly, and API should be able to add objects or properties without it breaking any requesting applications.  Only when an object or property is being removed, or changed would a new API version be required.

Any requestor of the API will need a method for passing the requested version as part of the request.  This is usually done in either the HTTP header, querystring, or URL.  Swagger/Open API requires the version to be passed in the URL because there is no way to expose different object contracts on the same HTTP action.  The Swagger/Open API UI and document are always specific to a single version with that version identified in the URL.  Example:

```http
https://edhpoc3.azurewebsites.net:443/swagger/docs/v1
```

## References
* [Microsoft API Guidelines](https://github.com/Microsoft/api-guidelines)
* [Microsoft API Versioning Guidelines](https://github.com/Microsoft/api-guidelines/blob/vNext/Guidelines.md#12-versioning)
* [Versioning an API With ASP.Net and swagger](https://medium.com/@MAliBazzi/versioning-an-api-with-asp-net-and-swagger-the-unusual-way-613e34e2d61a)
* [API Versioning in ASP.Net Core with Nice Swagg](https://blog.jimismith.me/blogs/api-versioning-in-aspnet-core-with-nice-swagg)
