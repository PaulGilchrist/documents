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

It is important that these OData parameters are used in combinations to return the absolute minimum data required by the requesting application to ensure the lowest bandwidth consumption, latency, and cost while simultaneously supplying the highest performance and scalability.  Below is an example of how these can be combined together in complex ways:

```http
odata/users?$select=id,firstName,lastName&$filter=status eq ‘active’&$top=10&$skip=0&$expand=addresses($select=zipCode;$filter=status eq ‘active’;$top=1)
```

## References

* [odata.org]( https://www.odata.org/)
