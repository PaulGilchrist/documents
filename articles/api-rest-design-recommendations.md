# API REST Design Recommendations

## Executive Summary & Scope

A good API design contain business rules, and security laying the foundation for future developed applications to focus on data presentation and the user experience.  The initial API development effort will be many times offset by the re-use, consistency, and single source of the truth it provides to new development efforts, while simultaneously decoupling the actual system of record details.

The API layer will be Internet facing, and securely accessible from any device and any location, opening up flexibility not only for future application development, but also when integrating with partners and customers.  This document will discuss the RESTful API best practices, defined constraints and the advantages they bring.  The stateless server constraint for example, means it does not matter what server is servicing the request when no state needs to be shared across servers.  This allows for linear scaling of the web layer, and simplification of the load balancing layer.   Swagger/OpenAPI is a requirement for discovery allowing self-documentation and discovery for cloud hosting and other automated environments. 

Bandwidth efficiencies are observed through lazy loading of data only when needed by the client, and the non-data, presentation layer of an application being cacheable for a longer period due to its all static content.  For example, a panel can request data from the API only when it is opened by the user. In a responsively designed application, smaller screen formats may decide to not pull down data of lower priority or for sections of a page that are not displayed on due to size constraints.


## API Goals

The purpose of an API is to make securely available and business rule enforced data in the smallest format required by the client.  An example of an enforced business rule would be automatic filtering of returned communities to only those a user has rights to see.  Capabilities to reduce the returned data size should be incorporated into any good API design.  Some of these capabilities would include allowing the client to select just the fields they want to return.  Other examples would be allowing for search criteria or page size criteria to reduce the result set further.  These and other capabilities ensure the most efficient use of network bandwidth, and client memory.

The purpose of an API is not to process, or aid in presenting the data on behalf of the client.  Things like sorting, or de-normalizing the data for presentation (ex: FullName from first and last) should be handled at the client for both reduced network traffic, and linear CPU scaling.  For an API accessed by thousands of clients, it is much better to have each client sort the data once, than the server be requested to sort the data thousands of times.

A properly designed client is also expected to handle caching taking hints supplied from the API.  For example some data changes are infrequent such as a division name and address and are safe to cache for longer periods on the client.  If the client uses division name information on several pages, it is expected to cache this information locally and not keep requesting the same information from the API.

The API features discussed in the remainder of this document are focused on these overall goals.

## Technologies Discussed

* Endpoint = Microsoft ASP.Net MVC OData and standard controllers
* Security = OAuth 2.0 identity token with claims processed by a Microsoft OWIN pipeline
* Documentation & Discovery = Swagger/OpenAPI
* Object Relational Mapping (ORM) = Microsoft Entity Framework
* Protocol = Open Data Protocol (OData) and standard REST

## Semantics

* All API endpoints should be named using plural nouns, not verbs or names denoting behavior.  Avoiding thinking in terms of RPC like method names.  HTTP will pass the action verb as a GET, POST, PUT, or DELETE to the URL location of the object (noun)
   * Bad Examples: `getAccounts, createCommunity, or updateGroup`
   * Good Examples: `accounts, communities, groups`
* Version information will not be in the endpoint URL.  The server will return the latest version of any resource unless the HTTP Accept header for the request includes version information.  **<- Needs revision**
   * Test Driven Development helps with versioning so as a new version is introduced, the old version scripts still exists to ensure nothing breaks.
* Must return data in JavaScript default of camelCase and not the .Net format of PascalCase.  .Net can achieve this automatically when configured as follows:

```cs
var jsonFormatter = config.Formatters.OfType<JsonMediaTypeFormatter>().FirstOrDefault();
jsonFormatter.SerializerSettings.ContractResolver = new CamelCasePropertyNamesContractResolver();
```

* Do not use underscores for property or function names as they are unconventional for JavaScript
* Resources containing timestamps should ensure the time zone is UTC and the format follows the ISO 8601 standard.  Example: `2014-12-01T18:02:24.343Z`
   * JSON.Net will default to ISO 8601, but the following line will be needed in `Application_Start` to ensure consistent UTC timezone:

```cs
GlobalConfiguration.Configuration.Formatters.JsonFormatter
SerializerSettings.DateTimeZoneHandling = DateTimeZoneHandling.Utc
```

* Consistent HTTP codes will be returned from all endpoints.  This can be handled by having all endpoints return an `HttpResponseMessage` or `IActionResult` object that includes not just the returned resource, but also additional headers and status information.
   * `200 (ok)` - on successful get, update or delete (not on post - see below)
   * `201 (created)` - on successful POST (also include fully qualified URL to the new object in http location header)
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

## Hypermedia as the engine of application state (HATEOAS)

* The base URL for the API will be [https://api.<company-name>.com](https://api.<company-name>.com) and all API endpoints will begin from this level rather than from the Web.API initial path of [https://api.<company-name>.com/api](https://api.<company-name>.com/api).  A single entry point can be supplied to the client application with all API endpoints discoverable from there.  The client application will not need hard coded knowledge of the API endpoint URLs
* Every resource obtained from the API will return a URL that points to itself.  This URL will be globally unique and used to identifiy the resource rather than returning an ID parameter.  This allows the flexibility for the API path to change when the new path is not hard coded in the client, but passed back from the API itself.
* Example of resource containing sub resources

```json
{
	"url": "https://api.<company-name>.com/users/1051",
	"firstName": "Paul",
	"lastName": "Gilchrist",
	"email": "paul.gilchrist@outlook.com",
	"addresses": [
		"https://api.<company-name>.com/users/1051/addresses/912",
		"https://api.<company-name>.com/users/1051/addresses/913",
		"https://api.<company-name>.com/users/1051/addresses/914"
	]
}
```

* Child objects should by default not be loaded into the parent object to reduce server load, and WAN traffic requirements.  Child objects will be sent as URL references (see above example) allowing the client application to discover and access them as needed.
* A POST method will return location http header containing the URL to the new resource.  This method is preferred to just returning the ID of the newly created object, and allows the client to optionally call the URL later for GET, PUT, or DELETE methods.
* In addition to the self-documenting nature of hypermedia, the API will also enable .Net’s auto-generated help pages to describe each endpoint as user viewable HTML pages.

## Behavior

* GET, PUT, and DELETE methods must be Idempotent (Regardless of times called, the result will be the same)
   * POST does not need to be Idempotent
* Use a `$filter` parameter on larger GET requests to support conditional expressions for what data should be returned.  The `$filter` parameter wis URL friendly using common syntax like eq (equals), ne (not equals), gt (greater than), lt (less than), ge (greater or equal to), le (less than or equal to).
   * Conditions would be combined together using “and’ and ‘or’ keywords
   * Conditions would support keywords like ‘true’, ‘false’, and ‘null’
   * Partial match conditions would be supported such as `Contains()`, `StartsWith()`,`EndsWith()`, `DateTime()`.
      * Examples: Any of the three examples `FirstName.StartsWith("Pa")`, `FirstName.Contains(‘au’)`, or `FirstName.EndsWith("ul")` would all return the name ‘Paul’ as well as other matches meeting the condition.
      * DateTime Example:  `OrderDate gt DateTime(2015, 1, 1)`
      * System.LINQ.Dynamic can achieve all of the above capabilities
* Use `$top` and `$skip` parameters on larger GET requests to reduce result set size when needed.  This is to be implemented on a case by case basis and should not be expected behavior for all endpoints.
   * Default and maximum page sizes should be defined in the `web.config` file `AppSettings` to prevent excessively large return sizes.
   * Ex: `GET https://api.<company-name>.com/users/?$top=10&$skip=20` gets page 3
* Use a `$select` parameter on larger GET requests to reduce the properties returned in the response.  This is useful for example when wanting to get a list of object names for populating a dropdown rather than getting the full object details.
   * Ex: `GET https://api.<company-name>.com/users/?$select=id,firstName,lastName`
   * JSON.Net will do this during serialization as long as the model being returned contains ShouldSerialize functions for each property.
* Any long duration resources should return with an `Expires` HTTP header set.  (example: tip of the day with 24 hours expire)
   * Set `Cache-Control` header when setting a duration (like `max-age=3600` seconds) versus an exact expiration time.  `s-maxage` should be set for proxy server cache hints reducing load on API server.

## Association Examples

The following examples will show how Create, Read, Update and Delete (CRUD) commands are issued for the 0 or 1 to 1, 1 to many, and many to many object relationship types.  It is assumed that the resources have already been created before the association, and the application has a href to each resource being associated.

## Many to Many

In the example below, an address is being associated to a user, but in a many to many relationship associations can be done in either direction making `.../addresses({addressid})/users` equally supported.

   `POST .../users({userid})/addresses`
   * Request body contains `{ href: ".../users({userid})/addresses({id})" }`
   * Creates an association between user `{userid}` and address `{id}`
   * Response is `201 (created)` with `.../users({userid})/addresses({id})` in the message body and the location header.
   * If association already existed, response is `304 (not modified)`

   `DELETE .../users({userid})/addresses({id})`
   * Deletes the association between user `{userid}` and address `{id}` but does not delete the address resource.
   * Response is `200 (ok)` with empty message body
   * If association did not previously exist, response is `404 (not found)`

   `GET .../users({userid})/addresses`
   * Returns all addresses associated with the user `{userid}`.

   `GET .../users({userid})/addresses({id})`
   * Redirects the request to `.../addresses({id})` which is the href the address will always use when referring to itself

   `PUT .../users({userid})/addresses({id})`
   * Not supported as it is equivalent to PUT `.../addresses({id})` which is always the href returned when referring to itself

## One to Many
	
In the example below, a division is being associated to an area, unlike the many-to-many example, this associations can be done only in the one direction.

`POST .../areas({areaId})/divisions`

   * Request body contains an href to `{ href: ".../areas({areaId})/divisions({id})" }`
   * Creates an association between area `{areaId}` and division `{id}`
   * Response is `201 (created)` with `.../areas({areaId})/divisions({id})` in the message body and the location header.
   * If association already existed, response is `304 (not modified)`

`DELETE .../areas({areaId})/divisions({id})`

   * Deletes the association between area `{divisionid}` and division `{id}` but does not delete the community resource.
   * Response is `200 (ok)` with empty message body
   * If association did not previously exist, response is `404 (not found)`

`GET .../areas({areaId})/divisions`

   * Returns all divisions associated with the area `{areaId}`.

`GET .../areas({areaId})/divisions({id})`

   * Redirects the request to `.../divisions({id})` which is the href the division will always use when referring to itself

`PUT .../areas/{areaId}/divisions({id})`

   * Not supported as it is equivalent to PUT `.../divisions({id})` which is always the href a division uses when referring to itself

## Zero or One to One
	
In the example below, a user is being associated to a userDetail.
 
`POST .../users({userid})/userdetail`
   * Request body contains a complete userDetail resource 
   * If the user does not already have a userDetail associated, it creates a new userDetail and responds with a `201 (created)` with `.../users({userid})/userdetail` in the message body and the location header.
   * If a userDetail is already associated with user `{userid}`, then response is `409 (conflict)`, and a userDetail resource is not created.  Details will be added to the response body telling the client that a userDetail for user `{userid}` already existed, and give a href to the existing resource in the location header and body to be used should they want to PUT updates to the existing userDetail resource.

`DELETE .../users({userid})/userdetail`
   * Deletes the association between user {userid} and userdetail and also deletes the userdetail resource.
   * Response is 200 (ok) with empty message body
   * If association did not previously exist, response is 404 (not found)

`GET .../users({userid})/userdetail`
   * Returns the one userdetail resource associated with the user {userid}.

`GET .../users({userid})/addresses({id})`
   * Not supported as the {id} is never revealed to the client in a zero or one to one relationship

`PUT .../users({userid})/userdetail`
   * Request body contains a complete userDetail resource.
   * If the user does not already have a userDetail associated, the response is a `404 (not found)` with an empty message body.
   * If a userDetail is already associated with user `{userid}`, then response is `200 (ok)`, and the update is completed.


## Security

* Some endpoints in the API will support Anonymous access, so security filters should be used at the endpoint level, and not defined globally across the API.
* The server should maintain no session state and expect an OAuth 2.0 JWT bearer token to be supplied with any request requiring authentication.
* SSL should be used across the entire API including anonymous endpoints.  This is required to protect both the data moving between the client and server, and the tokens attached to the requests.  Keep in mind that SSL will eliminate the use of caching appliances between the server and client, making client caching all the more important, or an API specific appliance can be used to decrypt and cache responses.

## Client Expectations

* Clients must always set the http `Accept` header to tell the server what result format is supported, in priority order with recommended format listed first (usually `application/json`)
      * Can use multiple separated by semicolon (ex: `application/json;application/xml`), but xml should be out of scope.
      * The server should always set the http `Content-Type` header to tell the client what result format is being returned (usually `application/json`)
* Client must set the HTTP `media-type` header if requesting an older version of a resource than the current version.
* A POST method will return `location` http header containing the URL to the new resource.  This method is preferred to just returning the ID of the newly created object, and allows the client to optionally call the URL later for GET, PUT, or DELETE methods.
* Client must pass an HTTP `authentication` header in every call requiring authorization.  The client is expected to handle token requests and renewals as the server will return a `401 (unauthorized)` or `403 (forbidden)` for any invalid token or insufficient role membership respectfully.

## Infrastructure

* Short DNS TTL going to multiple CName endpoints with weights and health checks, allows for an API to be hosted from multiple datacenters simultaneously
* A `Health` endpoint should be developed that will return information to monitoring or logging tools regarding the health of the API and its components.  SCOM or other tools can access this specific endpoint to confirm availability of both the API and backend database.

## Visual Studio and .Net Specific Implementation Recommendations

* Put the API into its own project separate from the Entity Framework data project to maintain separation of concerns.
* The API should access data through Repositories.  To support OData, the repositories should return IQueryable and not IEnumerable objects so OData can manipulate the query before it is executed against Entity Framework.
   * For .Net full framework, `Ninject.MVC5` and `WebApiContrib.ioc.Ninject` can be used for dependency injection into the controllers.  Direct Injection is built into .Net Core eliminating the need for Ninject or similar packages.
      * Ninject.CreateKernel:

```cs
GlobalConfiguration.Configuration.DependencyResolver = new NinjectResolver(kernel);
```

* Ninject.RegisterServices:

```cs
kernel.Bind<IApiRepository>().To<ApiRepository>();
kernel.Bind<ApiContext>().To<ApiContext>();
```

   * The API layer can translate the model from data specific to API specific and back again passing them through the repositories.
* Do not just use a single DbContext and Repository for the entire database but rather bounded contexts to different areas of data when nesting of related data ($expand) is not needed.  This keeps each context more manageable and the application more scalable.  A single database can still be used, and single tables can be accessible through multiple contexts.  Example: Customer table would be in both the sales and warranty contexts.
   * This is not alweays possible when using OData as its $expand capability can not cross DbContexts, and is a key capability for performant nested data access.

## Notes

* Validation caching can be used (ETag) so client can send server back the ETag with `if-none-match` header, and server will only return resource if it has changed (different ETag at server).  Server will return status `304 Not Modified` if content has not changed, or a `200 OK` with full return JSON if content has changed.  DB last modified date makes a good ETag.  `Last-Modified` is also a valid HTTP header to completely replace ETag.
* Put solution and recommendation together around `Expires` header

## Action Items

- [ ] Revise section on versioning
- [ ] Include more around OData
- [ ] Async vs Sync when calling across services
- [ ] Update Behavior section to discuss OData implementation
