# API Caching

## Summary
This document discusses the issues and solutions around caching an API including complex query API’s such as OData while supporting all current browser and routing appliance caching behavior.  This document also discusses a standard method for cache busting that works across all HTTP requests.

## Problems to Address

* If you use the HTTP headers `Cache-Control` or `Expires`, the data can be cached by any appliance between the server and client for that time period and any new request may not make it to the server eliminating the ability to cache bust unless the URL is changed.
* If changing the URL as a method of cache busting, the URL cannot be used as the memory cache key without first detecting and removing the change made to the URL or data duplication will exist and grow within the memory cache.
* Once something is placed in the memory cache, it can be retrieved, but that retrieval will not contain the amount of time remaining before expiration.  To fix this, the new object needs to be created that contains the original object, and the AbsoluteExpiration time.  This new object is then what is stored in MemoryCache with the key.
 
## API Cache Usage Suggestion

* API endpoint will first use the entire URL including querystring as the MemoryCache "key" to determine if the data should be returned from cache or the original source
* If fresh data is retrieved from the original source, that data will be added to the memory cache using the above mentioned key.
* Anything added to MemoryCache will be given an `AbsoluteExpiration` based on the object type.  *Where should these expiration times be defined? Web.config?*
* The `AbsoluteExpiration` will also be set on the response's `Cache-Control: s-maxage=<seconds>` header
* Make sure to also read over the scenario where the client has specifically requested to bust the current cached data.

## Cache Busting

* Add `nonce=<GUID>` to the http request querystring
* Server will not find URL in memory cache so will get fresh data from original source
* Before storing new data into cache, server must check for nonce querystring parameter and do the following if it is found:
   * Create memory cache key consisting of the full URL with the nonce parameter removed.  This should match any original request made that might be found in the memory cache.  Use this new key for the remaining steps below
   * Remove key/value from memory cache if found
   * Add new data to cache using new key and setting the appropriate `AbsoluteExpiration`
* The `AbsoluteExpiration` will also be set on the response's `Cache-Control: s-maxage=<seconds>` header
 
## Code Example

```cs
[HttpGet]
[Route("cacheDemo", Name = "CacheDemo")]
public HttpResponseMessage Get() {
	//Anything added to MemoryCache will be given an AbsoluteExpiration based on the object type.
	//For this demo we will use 60 seconds but this information should come from another source in production like web.config or SQL to memoryCache
	int maxAgeSeconds = 60;
	DateTimeOffset absoluteExpiration;
	string data = null;
	// Use the entire URL including querystring as the MemoryCache "key" to determine if the data should be returned from cache or the original source
	// If nonce=<GUID> is sent in querystring, then "key" will never be found in cache (use this method to bust cache)
	var key = Request.RequestUri.AbsoluteUri.ToLower();
	MemoryCache memoryCache = MemoryCache.Default;
	CacheValue cacheValue = (CacheValue)memoryCache.Get(key);
	if(cacheValue != null) {
		data = (string)cacheValue.Data;
		absoluteExpiration = cacheValue.AbsoluteExpiration;
		// For demo purposes we are going to change what was in the cache to a new result so the caller can see the cache as source
		data = "Old data from cache";
	} else {
		// Simulate new data coming from the original source
		data = "New data from original source";
		absoluteExpiration = DateTimeOffset.UtcNow.AddSeconds(maxAgeSeconds);
		// Before storing new data into cache, check for nonce querystring parameter
		if(key.Contains("nonce=")) {
		// Key should match any original request that would be made without cache busting.
		key = RemoveQueryStringByKey(key, "nonce");
		// Remove the old key/value from memory cache if found
		memoryCache.Remove(key);
		}
		memoryCache.Add(key, new CacheValue() { Data = data, AbsoluteExpiration = absoluteExpiration }, absoluteExpiration);
	}
	HttpResponseMessage response = Request.CreateResponse(HttpStatusCode.OK, data);
	response.Headers.CacheControl = new CacheControlHeaderValue() { MaxAge = absoluteExpiration.Subtract(DateTime.Now) };
	return response;
}

private string RemoveQueryStringByKey(string url, string key) {
	var uri = new Uri(url);
	// This gets all the query string key value pairs as a collection
	var newQueryString = HttpUtility.ParseQueryString(uri.Query);
	// This removes the key if exists
	newQueryString.Remove(key);
	// This gets the page path from root without QueryString
	string pagePathWithoutQueryString = uri.GetLeftPart(UriPartial.Path);
	return newQueryString.Count > 0
		? String.Format("{0}?{1}", pagePathWithoutQueryString, newQueryString)
		: pagePathWithoutQueryString;
}

public class CacheValue {
	public object Data { get; set; }
	public DateTimeOffset AbsoluteExpiration { get; set; }
}
```

## Default OData Response (non-cached endpoints)

```http
Cache-Control: no-cache
Expires: -1
Pragma: no-cache
```
