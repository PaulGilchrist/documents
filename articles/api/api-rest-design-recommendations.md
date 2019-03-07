# API REST Design Recommendations

## Executive Summary & Scope

A good API design contain business rules and security, laying the foundation for future developed applications to focus on data presentation and the user experience.  The initial API development effort will be many times offset by the re-use, consistency, and single source of the truth it provides to new development efforts, while simultaneously decoupling the actual system of record details.

The API layer should be Internet facing, and securely accessible from any device and any location, opening up flexibility not only for future application development, but also when integrating with partners and customers.  This document will discuss the RESTful API best practices, defined constraints and the advantages they bring.  The stateless server constraint for example, means it does not matter what server is servicing the request when no state needs to be shared across servers.  This allows for linear scaling of the web layer, and simplification of the load balancing layer.   Swagger/OpenAPI is a requirement for discovery allowing self-documentation and discovery for cloud hosting and other automated environments.

Bandwidth efficiencies are observed through lazy loading of data only when needed by the client, and the non-data, presentation layer of an application being cacheable for a longer period due to its all static content.  For example, a panel can request data from the API only when it is opened by the user. In a responsively designed application, smaller screen formats may decide to not pull down data of lower priority or for sections of a page that are not displayed on due to size constraints.

## API Goals

The purpose of an API is to make securely available and business rule enforced data in the smallest format required by the client.  An example of an enforced business rule would be automatic filtering of returned divisons to only those a user has rights to see.  Capabilities to reduce the returned data size should be incorporated into any good API design.  Some of these capabilities would include allowing the client to select just the fields they want to return.  Other examples would be allowing for search criteria or page size criteria to reduce the result set further.  These and other capabilities ensure the most efficient use of network bandwidth, and client memory.

The purpose of an API is not to process, or aid in presenting the data on behalf of the client.  Things like sorting, or de-normalizing the data for presentation (ex: FullName from first and last) should be handled at the client for both reduced network traffic, and linear CPU scaling.  For an API accessed by thousands of clients, it is much better to have each client sort the data once, than the server be requested to sort the data thousands of times.

A properly designed client is also expected to handle caching, taking hints supplied from the API.  For example some data changes are infrequent such as a division name and address and are safe to cache for longer periods on the client.  If the client uses division name information on several pages, it is expected to cache this information locally and not keep requesting the same information from the API.

The API features discussed in the remainder of this document are focused on these overall goals.

## Technologies Discussed

* Endpoint = Microsoft ASP.Net MVC OData and standard controllers
* Security = OAuth 2.0 identity token with claims processed by a Microsoft OWIN pipeline
* Documentation & Discovery = Swagger/OpenAPI
* Object Relational Mapping (ORM) = Microsoft Entity Framework
* Protocol = Open Data Protocol (OData) and standard REST

## Semantics

* All API endpoints should be named using plural nouns, not verbs or names denoting behavior.  Avoiding thinking in terms of RPC like method names.  HTTP will pass the action verb as a GET, POST, PUT, PAYCH, or DELETE to the URL location of the object (noun)
   * Bad Examples: `getAccounts, createCommunity, or updateGroup`
   * Good Examples: `accounts, communities, groups`
   * Test Driven Development helps with versioning so as a new version is introduced, the old version scripts still exists to ensure nothing breaks.
* Must return data in the JavaScript default of camelCase and not the .Net format of PascalCase.  ASP.Net can achieve this automatically when configured as follows:

.Net Full Framework

```cs
var jsonFormatter = config.Formatters.OfType<JsonMediaTypeFormatter>().FirstOrDefault();
jsonFormatter.SerializerSettings.ContractResolver = new CamelCasePropertyNamesContractResolver();
```

.Net Core Framework

```cs
services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_1)
   .AddJsonOptions(options => {
      options.SerializerSettings.ContractResolver = new Newtonsoft.Json.Serialization.CamelCasePropertyNamesContractResolver();
   });
```

* Do not use underscores for property or function names as they are unconventional for JavaScript
* Resources containing timestamps should ensure the time zone is UTC and the format follows the ISO 8601 standard.  Example: `2014-12-01T18:02:24.343Z`
   * JSON.Net will default to ISO 8601, but the following line will be needed in `Application_Start` to ensure consistent UTC timezone:

.Net Full Framework

```cs
GlobalConfiguration.Configuration.Formatters.JsonFormatter
SerializerSettings.DateTimeZoneHandling = DateTimeZoneHandling.Utc
```

.Net Core Framework

```cs
services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_1)
   .AddJsonOptions(options => {
      options.SerializerSettings.DateTimeZoneHandling = DateTimeZoneHandling.Utc;
});
```

* Consistent HTTP codes will be returned from all endpoints.  This can be handled by having all endpoints return an `HttpResponseMessage` or `IActionResult` object that includes not just the returned resource, but also additional headers and status information.
   * `200 (ok)` - on successful get, update or delete (not on post - see below)
   * `201 (created)` - on successful POST (also include fully qualified URL to the new object in http location header)
   * `202 accepted` - returned for a long running process to inform the requestor that the request has been accepted.  There will always be some form of follow-up notification when the process completes.  This is sometimes handled by a callback webhook, or by a separate status API endpoint and correlation ID.
   * `204 (no content) - returned when the action was successful, but there is no data to return.  This is common for DELETE or association POST requests.
   * `304 (not modified)` – returned if the client already has the most recent copy of the resource.  Usually an ETag is sent to the server using the `If-None-Match` header and if the server has the same information, it sends back a 304 instead of the resource.  Entity Framework does not have built is support for ETags as it would break the stateless constraint.  There is an NuGet package named CacheCow.Server with its associated SQL extension that adds this functionality by storing ETags in SQL server, but this add an extra DB hit for each update, so it may be best to not rush into server caching, and focus on best practices around client side caching.
   * `400 (bad request)` - when a resource is submitted but the data is not complete or in the correct format for the requested POST or PUT operation.
   * `401 (unauthorized)` - when user or application is not authenticated.  This is handled automatically by the OWIN layer along with the `[Authorize]` filters set on each endpoint.
   * `403 (forbidden)` - when user and application are authenticated but not authorized to execute the particular behavior on the given resource.  This is handled automatically by the OWIN layer along with the `[Authorize]` filters set on each endpoint.
   * `404 (not found)` - when a URL to a non-existent resource is used.
   * `409 (conflict)` - when there is a server issue (like SQL offline) but the request is re-try able
   * `412 (precondition failed)` - A PUT request would use the `If-match` header so an update is made only if the resource did not change since the user last requested it.  The server compared the ETag in the `If-Match` with what is has for the resource to determine if the client has the latest copy, returning a 412 if it does not.
* If detailed information is known about a failed request, that information can be returned in the HTTP body with information such as follows:
   * `code` = more application specific error code
   * `message` = error message that can be shown directly to the end user
   * `developerMessage` = detailed error message meant for the developer and not to be shown directly to the end user
   * `moreInfo` for a URL to more information

## Behavior

* GET, PUT, and DELETE methods must be Idempotent (Regardless of times called, the result will be the same)
   * POST does not need to be Idempotent
* Use a `$filter` parameter on larger GET requests to support conditional expressions for what objects should be returned.  The `$filter` parameter should be URL friendly using common syntax like eq (equals), ne (not equals), gt (greater than), lt (less than), ge (greater or equal to), le (less than or equal to), etc..
   * Conditions should be combined together using “and’ and ‘or’ keywords
   * Conditions should support keywords like ‘true’, ‘false’, and ‘null’
   * Partial match conditions should be supported such as `contains()`, `startswith()`,`endswith()`, `length()`, etc..
      * Examples: Any of the three examples `firstName.startswith('Pa')`, `firstName.contains('au')`, or `firstName.endswith('ul')` would all return the name 'Paul' as well as other matches meeting the condition.
      * DateTime Example:  `orderDate gt datetime(2015, 1, 1)`
      * Tools like OData, or System.LINQ.Dynamic can achieve all of the above capabilities
* Use `$top` and `$skip` parameters on larger GET requests to reduce result set size when needed.  This is to be implemented on a case by case basis and should not be expected behavior for all endpoints.
   * Default and maximum page sizes should be defined in the `web.config` or `appSettings.json` files to prevent excessively large return sizes.
   * Ex: `GET https://api.company-name.com/users/?$top=10&$skip=20` gets page 3
* Use a `$select` parameter on larger GET requests to reduce the properties returned in the response.  This is useful for example when wanting to get a list of object names for populating a dropdown rather than getting the full object details.
   * Ex: `GET https://api.company-name.com/users/?$select=id,firstName,lastName`
   * JSON.Net will do this during serialization as long as the model being returned contains ShouldSerialize functions for each property.
* Any long duration resources should return with an `Expires` HTTP header set.  (example: tip of the day with 24 hours expire)
   * Set `Cache-Control` header when setting a duration (like `max-age=3600` seconds) versus an exact expiration time.  `s-maxage` should be set for proxy server cache hints reducing load on API server.

## Association Examples

The following examples will show how Create, Read, Update and Delete (CRUD) commands are issued for the 0 or 1 to 1, 1 to many, and many to many object relationship types.  It is assumed that the resources have already been created before the association, and the application has a href to each resource being associated.

## Many to Many

In the example below, an address is being associated to a user, but in a many to many relationship associations can be done in either direction making `.../addresses({addressid})/users` equally supported.

   `POST .../users({userId})/addresses({addressesId})`
   * Creates an association between user `{userId}` and address `{addressesId}`
   * Response is `204 (no content)` with empty message body
   * If association already existed, response is `304 (not modified)`

   `DELETE .../users({userId})/addresses({addressesId})`
   * Deletes the association between user `{userId}` and address `{id}` but does not delete the address object.
   * Response is `204 (no content)` with empty message body
   * If association did not previously exist, response is `404 (not found)`

   `GET .../users({userId})/addresses`
   * Returns all addresses associated with the user `{userId}`.

   `GET .../users({userId})/addresses({addressesId})`
   * Redirects the request to `.../addresses({id})` which is the href the address will always use when referring to itself

   `PUT .../users({userId})/addresses({addressesId})`
   * Not supported as it is equivalent to PUT `.../addresses({addressesId})` which is always the href returned when referring to itself

## One to Many
	
In the example below, a division is being associated to an area, unlike the many-to-many example, this associations can be done only in the one direction.

`POST .../areas({areaId})/divisions({divisionId})`

   * Creates an association between area `{areaId}` and division `{id}`
   * Response is `204 (no content)` with empty message body
   * If association already existed, response is `304 (not modified)`

`DELETE .../areas({areaId})/divisions({divisionId})`

   * Deletes the association between area `{areaId}` and division `{divisionId}` but does not delete the division object.
   * Response is `204 (no content)` with empty message body
   * If association did not previously exist, response is `404 (not found)`

`GET .../areas({areaId})/divisions`

   * Returns all divisions associated with the area `{areaId}`.

`GET .../areas({areaId})/divisions({divisionId})`

   * Redirects the request to `.../divisions({divisionId})` which is the href the division will always use when referring to itself

`PUT .../areas/{areaId}/divisions({divisionId})`

   * Not supported as it is equivalent to PUT `.../divisions({divisionId})` which is always the href a division uses when referring to itself

## Zero or One to One

In the example below, a user is being associated to a userDetail.

`POST .../users({userId})/userdetail`
   * Request body contains a complete userDetail resource 
   * If the user does not already have a userDetail associated, it creates a new userDetail and responds with a `201 (created)` with the new object in the message body and the location to the userDetail object in the header.
   * If a userDetail is already associated with user `{userId}`, then response is `409 (conflict)`, and a userDetail resource is not created.  Details will be added to the response body telling the client that a userDetail for user `{userId}` already existed, and give a href to the existing resource in the location header and body to be used should they want to PUT updates to the existing userDetail resource.

`DELETE .../users({userId})/userdetail`
   * Deletes the association between user {userId} and userdetail and also deletes the userdetail resource.
   * Response is 200 (ok) with empty message body
   * If association did not previously exist, response is 404 (not found)

`GET .../users({userId})/userdetail`
   * Returns the one userdetail resource associated with the user {userid}.

`GET .../users({userId})/addresses({userDetailId})`
   * Not supported as the {id} is never revealed to the client in a zero or one to one relationship

`PUT .../users({userId})/userdetail`
   * Request body contains a complete userDetail resource.
   * If the user does not already have a userDetail associated, the response is a `404 (not found)` with an empty message body.
   * If a userDetail is already associated with user `{userId}`, then response is `200 (ok)`, and the update is completed.

## Security

* Some endpoints in the API will support anonymous access, so security filters should be used at the endpoint level, and not defined globally across the API.
* The server should maintain no session state and expect an OAuth 2.0 JWT bearer token to be supplied with any request requiring authentication.
* SSL should be used across the entire API including anonymous endpoints.  This is required to protect both the data moving between the client and server, and the tokens attached to the requests.  Keep in mind that SSL will eliminate the use of caching appliances between the server and client, making client caching all the more important, or an API specific appliance can be used to decrypt and cache responses.

## Client Expectations

* Clients must always set the HTTP `Accept` header to tell the server what result format is supported, in priority order with recommended format listed first (usually `application/json`)
      * Can use multiple separated by semicolon (ex: `application/json;application/xml`), but xml should be out of scope.
      * The server should always set the HTTP `Content-Type` header to tell the client what result format is being returned (usually `application/json`)
* A POST method will return the full object in the body, and the `location` HTTP header containing the URL to the new resource.  This method is preferred to just returning the ID of the newly created object, and allows the client to optionally call the URL later for GET, PUT, or DELETE methods.
* Client must pass an HTTP `authentication` header in every call requiring authorization.  The client is expected to handle token requests and renewals as the server will return a `401 (unauthorized)` or `403 (forbidden)` for any invalid token or insufficient role membership respectfully.

## Infrastructure

* Short DNS TTL going to multiple CName endpoints with weights and health checks, allows for an API to be hosted from multiple datacenters simultaneously
* A `Health` endpoint should be developed that will return information to monitoring or logging tools regarding the health of the API and its components.  SCOM, Azure Traffic Manager, similar load balancers, API management tools, etc., can access this specific endpoint to confirm availability of both the API and backend database.

## Visual Studio and .Net Specific Implementation Recommendations

* All calls made between remote services like between the API and database services, should be made asyncronously.  This is very important and should not be overlooked.
* Put the API into its own project separate from any frint-end application code,  maintaining separation of concerns.
* The API should access data through Repositories.  To support OData, the repositories should return IQueryable and not IEnumerable objects so OData can manipulate the query before it is executed against Entity Framework.
   * For .Net full framework, `Ninject.MVC5` and `WebApiContrib.ioc.Ninject` can be used for dependency injection into the controllers.  Direct Injection is built into .Net Core eliminating the need for Ninject or similar packages.

.Net Full Framework

```cs
GlobalConfiguration.Configuration.DependencyResolver = new NinjectResolver(kernel);
```

* Ninject.RegisterServices:

```cs
kernel.Bind<IApiRepository>().To<ApiRepository>();
kernel.Bind<ApiContext>().To<ApiContext>();
```

.Net Core Framework

```cs
services.AddDbContext<ApiContext>(options => options.UseSqlServer(connection), ServiceLifetime.Singleton);
```

* Do not just use a single DbContext and Repository for the entire database but rather bounded contexts to different areas of data when nesting of related data ($expand) is not needed.  This keeps each context more manageable and the application more scalable.  A single database can still be used, and single tables can be accessible through multiple contexts.  Example: Customer table would be in both the sales and warranty contexts.
   * This is not alweays possible when using OData as its $expand capability can not cross DbContexts, and is a key capability for performant nested data access.

## Notes

* Validation caching can be used (ETag) so client can send server back the ETag with `if-none-match` header, and server will only return resource if it has changed (different ETag at server).  Server will return status `304 Not Modified` if content has not changed, or a `200 OK` with full return JSON if content has changed.  DB last modified date makes a good ETag.  `Last-Modified` is also a valid HTTP header to completely replace ETag.
* Put solution and recommendation together around `Expires` header

## References
* See [GitHub odate-core-template](https://github.com/PaulGilchrist/odata-core-template) for full source code of a working example of the above recomendations
* See [GitHub odate-core-template](https://github.com/PaulGilchrist/documents) for additional API best practice and training documentation