# API - OData Efficient Use of Query Capabilities

OData (Open Data Protocol) defines a set of best practices for building and consuming RESTful APIs.  OData supplies a minimal set of request query parameters that combine together to enable high levels of API endpoint re-usability and flexibility.

## OData supports the following request query parameters

* `$select` = Allows for returning a subset of properties.  This ensures bandwidth is not wasted returning properties that are not needed by the requestor.  A good example would be populating a `<select>` list with just names and id (`odata/users?$select=name,id`)
* `$filter` = Allows for returning a subset of objects.  This ensures bandwidth is not wasted returning objects that are not needed by the requestor.  A good example would be populating a `<table>`  with only active users (`odata/users?$filter=status eq ‘active’`)
* `$top` = Allows for returning a subset of objects not to exceed the given amount.  Example: `odata/users?$top=100` would return the first 100 objects or all objects if less than 100 exist
* `$skip` = When combined with $top, can be used for paging first skipping this many objects, then returning the amount given by $top.    Example: `odata/users?$top=10&$skip=20` would return page #3 with each page being 10 objects, and the first 20 objects being skipped.
* `$count` = Allows for returning the total count of how many objects would be returned based on any optional $filter.  If wanting to return just the count without the objects themselves, combine it with $top (`odata/users?$count=true&$top=0`)
* `$expand` = Allows for nesting child objects within their related parent objects.  This is a very effective way to return all needed data in the least amount of API calls.  Example: `odata/users?$expand=addresses` would return all user objects with each one having 0 or more nested address objects.  $expand can support multiple levels of nesting as in the example: ` odata/areas?$expand=divisions($expand=offices)`.  $expand also supports all other OData parameters at each expanded level contained within `()`, and separated by `;`
* `$orderBy` = Allows for server side sorting of the objects prior to their return.  It is strongly suggested to do all sorting at the client offload processing, and scaling linearly with the number of connected clients versus requiring the server to process all sorting requests on its static number of processors.  $orderBy can sort both ascending (asc) and decending (desc) as in the example `odata/users?$orderBy=lastName asc`.

## Best Practices

### Performance Best Practices

* ```Return no more data than is necessary``` - Combine OData parameters to return the absolute minimum data required by the requesting application ensuring the lowest bandwidth consumption, latency, and cost while simultaneously supplying the highest performance and scalability. 

  * Use $select to return only those columns that are needed in your query (applies to each $expand-ed level of the query)
  * Use $filter to return only those records that you need for your query (again, at every level)
  * Specify a multi-level $filter from the top level when you want to filter data in $expand-ed collections 
    * “$expand=financialVendor&$filter=financialVendor/marketId eq 114” works but “$expand=financialVendor($filter=marketId eq 114)” does not
  * If you only need the count, pass $count=true and $top=0
  * If you only need the $expand-ed data and nothing on the top level, just $select the expanded property.

Below is an example of how these can be combined together in complex ways:

```http
odata/users?$select=id,firstName,lastName&$filter=status eq ‘active’&$top=10&$skip=0&$expand=addresses($select=zipCode;$filter=status eq ‘active’;$top=1)
```

* ```Data changing infrequently should be cached at the client``` - Many data properties rarely change such as a user's name, email or phone number, or a division office's name or physical address.  Calling the API to re-retrieve this data each time it is needed, causes unnecessary load on both the client and API, as well as wasting network bandwidth.  In these scenarios, caching these objects is preferred.  The closer the cache is to the client browser the better for best linear scalability and reduction on shared service resource or bandwidth usage.  Time to live (TTL) and forced update options should be implemented to allow application configuration options for best cache control.

The below example shows support for caching data within an Angular service including support for TTL and forces refresh:

```JavaScript
    private users = new BehaviorSubject<User[]>([]);
    users$ = this.users.asObservable();
    private _lastUserDataRetreivalTime: number; // Time when user data was last retireved from the remote souce

    public getUsers(force: boolean = false): Observable<User[]> {
        if (force || this.users.getValue().length === 0 || Date.now() - this._lastUserDataRetreivalTime > environment.dataCaching.userData) {
            this.http.get<Address[]>(environment.apiUrl + 'addresses.json').pipe(
                retry(3),
                tap((users: User[]) => {
                    this._lastUserDataRetreivalTime = Date.now();
                    // Caller can subscribe to users$ to retreive the users any time they are updated
                    this.users.next(users);
                }),
                catchError(this.handleError)
            ).subscribe();
        }
        return this.users$;
    }
```

* ```Share data across client components``` - Some common data will be used across multiple client-side components such as user or organization names or other details.  There is no need to have each component re-query the API for this data when a single API request could be made, and all components share the response.  Client side frameworks such as Angular/RxJs even allow for notifying all consuming components when any of them refresh the shared data.
* ```Ensure appropriate database indexing exists for frequently used OData queries``` - Although OData allows the flexibility for a client to pass any $filter or $expand properties, there is a strong possibility that the improper use of these can cause slow backend database performance and unnecessary resource consumption.  The larger the total number of object records or deeper the $expand, the more significant this can be.  You can initially use the OData `$count=true&$top=0` properties without a $filter to determine the total amount of data currently in the backend database for any given object.  For larger tables, ensure the API's backend database has been optimized for the given $filter and $expand being requested, or make the appropriate changes to either the database or OData query and needed.  If accessing a remote API, a request may need to be made to that team to review their current indexes based on your given query to determine if any optimization opportunities exist.
* ```Call APIs in parallel where possible``` - Client-side frameworks such as Angular/RxJs have native capabilities for parallel API calls such as 'combineLatest', 'forkJoin', etc...  API calls should be made in parallel whenever one does not depend on the output of another.  This can have a significant improvement to the overall time to retrieve all needed data.  The 'network' tab from Chrome debugging tools is a great place to quickly notice serialized API calls.
* ```Make all remote calls asynchronously``` - All calls to your API should be made asynchronously as should calls from your API to any external service such as a backend database or to another remote API.  Making these calls asynchronously ensures your application does not block waiting for a response and is best able to continue to do other work while waiting for the remote service to respond.  This is frequently overlooked in code even when the developer understands and regularly follows this best practice.
* ```Keep API calls re-usable and loosely coupled``` - Even when proxying back to another remote API, it is always best to keep all API endpoints re-usable and loosely coupled.  This allows the client application to retain control of the OData properties passed through to the remote API, and reduces both the code required, and the work performed by the proxy API.  Basically, the proxy API simply passes along the request and responses without any serialization/deserialization, or model definitions.  Below is an example of a re-usable and loosely coupled proxy API to a backend API:

```JavaScript
        /// <summary>
        /// Query backend API for Divisions
        /// </summary>
        /// <remarks>
        /// </remarks>
        [HttpGet]
        public async Task<IActionResult> GetRemoteApiDivisions() {
            try {
                HttpClient client = new HttpClient();
                // Allow the client to pass any valid OData query parameters
                var request = AppSettings.RemoteApiBaseUrl + "divisions" + HttpContext.Request.QueryString;
                var response = await client.GetAsync(request);
                return StatusCode(Convert.ToInt32(response.StatusCode), response.Content.ReadAsStreamAsync().Result);
            } catch (Exception ex) {
                return StatusCode(500, ex.Message);
            }
        }
```

The next example is not re-usable or loosely coupled:

```JavaScript
        /// <summary>
        /// Query backend API for Divisions
        /// </summary>
        /// <remarks>
        /// </remarks>
        [HttpGet]
        public async Task<IActionResult> GetRemoteApiDivisions() {
            try {
                HttpClient client = new HttpClient();
                // Hard wire query parameters before calling the remote OData API
                var request = AppSettings.RemoteApiBaseUrl + "divisions?$select=id,name,type,isActive&$filter=isActive eq true&$count=true");
                var response = await client.GetAsync(request);
                return StatusCode(Convert.ToInt32(response.StatusCode), response.Content.ReadAsStreamAsync().Result);
            } catch (Exception ex) {
                return StatusCode(500, ex.Message);
            }
        }
```

* ```Leverage the linear scalability of the client``` - Server side processing does not scale linearly with the number of clients accessing the API.  Conversly, client side processing guarantees as the number of concurrent clients increase, the number of processors available to do work, also increases.  When doing things like sorting ($orderBy), take advantage of each client's dedicated processors rather than overworking the server making it handle all concurrent sorting requests.  This may not always be possible as in the case where the $orderBy is needed in combination with a $top to ensure the correct top X objects are returned.

### Implementation Best Practices

* ```Understand RESTful design best practices```
  * URLs - Plural nouns for endpoints
    * No verbs or behavioral names
    * Path from root or must match relationship hierarchy
    * Passed filters determine how many results are returned
      * ID for singular, OData for advanced query
  * HTTP actions (GET, POST, PUT, PATCH, DELETE)
    * Wrong: GetArea, CreateDivision, UpdateEmployee
    * Right: Areas, Divisions, Employees
  * GET, PUT, and DELETE methods must be Idempotent
    * Regardless of times called, the result will be the same
    * POST can also be idempotent through SQL uniqueness constraints and API trapping of those constraints and returning a 409 (conflict) error with details of what constraint failed
  * Human readable data
    * Wrong: status = 0, 1, 2, 3, 4, etc.
    * Right: status = pending, approved, rejected, complete, etc.
  * Property Names - camelCase objects and properties
    * No underscores or word separators as they are unconventional for JavaScript
  * Time - UTC following ISO 8601 standard
  * Stateless and tolerant of transient failures
    * Retry logic (built into EF)
    * Valuable for all cloud development not just REST APIs
  * Consistent HTTP codes with details in body
    * 200 (ok), 201 (created), 202 (accepted), 304 (not modified), 400 (bad request), 401 (unauthorized), 403 (forbidden), 404 (not found), 409 (conflict), 412 (precondition failed), etc.
* ```Understand OData and leverage for optimized requests```
  * Expand, Filter, Select, OrderBy, Top, Skip, Count
* ```Code for efficient bandwidth use and reduced costs```
  * OData Expand Example:
    * areas -> divisions -> branches -> employees
  * Caching: Client is better than App API which is better than remote API
    Can leverage ETag or change notifications if available
* ```Leverage Swagger for discovery```
  * Communicate to API team if discovery information is missing (ex: roles required)
* ```Application API’s should also use REST, OData, OAuth, and Swagger```
* ```Understand OAuth tokens and API keys```
  * Stateless means passed in header on every request
  * User to API = OAuth bearer token (with roles)
    * Implicit flow
  * API to API = API key (Azure app settings define roles)
  * Claims\Roles
* ```Understand cloud development best practices```
  * Asynchronous and parallel
  * Azure costs – client offloading, bandwidth usage reduction
    * Bandwidth free within an Azure regional datacenter
  * Latency – prefetching, caching, background workers
  * Make no assumptions about underlying infrastructure, location, IP, etc.
  * Let DevOps manage keys (Azure App Settings)
  * Leverage App Insights custom telemetry, and troubleshoot EDH there also

## References

* OData Specification and detailed documentation - [odata.org]( https://www.odata.org/)
* See document [REST Design Recommendations](https://github.com/PaulGilchrist/documents/blob/master/articles/api/api-rest-design-recommendations.md)
* See document [API - OData Setup for ASP.Net Core](https://github.com/PaulGilchrist/documents/blob/master/articles/api-odata-setup-for-dot-net-core.md) for proper implementation of OData
* See document [API - Swagger/Open API for ASP.Net Core using Swashbuckle](https://github.com/PaulGilchrist/documents/blob/master/articles/api-swagger-openapi-for-asp-net-core-using-swashbuckle.md) for proper configuration of OData controller function comments and annotation recomendations
* See [GitHub odate-core-template](https://github.com/PaulGilchrist/odata-core-template) for full source code of a working example of all above steps
