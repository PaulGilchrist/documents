# API - Throttling / Rate Limiting for ASP.NET Core

API throttling can easily be added to ASP.Net and ASP.Net Core using existing middleware NuGet packages.  Throttling allows you to control the rate of requests that a client can make to the API based on IP address, client API key, and request route.  Throttling can limit the number of calls allowed in a given period of time to either individual request routes, or across the whole API.  Specific IP address or client keys can optionally be whitelisted so they will not have throttling policies applied.  Similarly, specific IP address or client keys can optionally apply custom policies overriding global policies.

## ASP.NET Core Throttling Steps

1. Add NuGet package [AspNetCoreRateLimit](https://github.com/stefanprodan/WebApiThrottle)
* If using ASP.NET Full Framework, use the following guide [API - Throttling / Rate Limiting for ASP.NET Full Framework](https://github.com/PaulGilchrist/documents/blob/master/articles/api-throttling-rate-limiting-for-asp-net-full-framework.md)
2. In the file `Startup.cs` and function `ConfigureServices()` add the following code before `services.AddMvc()`

```cs
// needed to load configuration from appsettings.json
services.AddOptions();
// needed to store rate limit counters and ip rules
services.AddMemoryCache();
// load general configuration from appsettings.json
services.Configure<ClientRateLimitOptions>(Configuration.GetSection("ClientRateLimiting"));
// load client rules from appsettings.json
services.Configure<ClientRateLimitPolicies>(Configuration.GetSection("ClientRateLimitPolicies"));
//load general configuration from appsettings.json
services.Configure<IpRateLimitOptions>(Configuration.GetSection("IpRateLimiting"));
//load ip rules from appsettings.json
services.Configure<IpRateLimitPolicies>(Configuration.GetSection("IpRateLimitPolicies"));
// inject counter and rules stores
services.AddSingleton<IClientPolicyStore, MemoryCacheClientPolicyStore>();
services.AddSingleton<IRateLimitCounterStore, MemoryCacheRateLimitCounterStore>();
```

3. In the file `Startup.cs` and function `Configure()` add the following code before `app.UseMvc()`

```cs
loggerFactory.AddConsole(Configuration.GetSection("Logging"));
loggerFactory.AddDebug();
app.UseClientRateLimiting();
app.UseIpRateLimiting();
```

4. Add default throttling settings to appsettings.json adjusting the below example to fit your requirements:

```json
"ClientRateLimiting": {
   "EnableEndpointRateLimiting": false,
   "StackBlockedRequests": false,
   "ClientIdHeader": "X-ClientId",
   "HttpStatusCode": 429,
   "EndpointWhitelist": [ "get:/api/license", "*:/api/status" ],
   "ClientWhitelist": [ "dev-id-1", "dev-id-2" ],
   "GeneralRules": [
      {
         "Endpoint": "*",
         "Period": "1s",
         "Limit": 2
       },
       {
         "Endpoint": "*",
         "Period": "15m",
         "Limit": 100
       },
       {
         "Endpoint": "*",
         "Period": "12h",
         "Limit": 1000
       },
       {
         "Endpoint": "*",
         "Period": "7d",
         "Limit": 10000
       }
   ]
},
"IpRateLimiting": {
   "EnableEndpointRateLimiting": false,
   "StackBlockedRequests": false,
   "RealIpHeader": "X-Real-IP",
   "ClientIdHeader": "X-ClientId",
   "HttpStatusCode": 429,
   "IpWhitelist": [ "127.0.0.1", "::1/10", "192.168.0.0/24" ],
   "EndpointWhitelist": [ "get:/api/license", "*:/api/status" ],
   "ClientWhitelist": [ "dev-id-1", "dev-id-2" ],
   "GeneralRules": [
      {
         "Endpoint": "*",
         "Period": "1s",
         "Limit": 2
      },
      {
         "Endpoint": "*",
         "Period": "15m",
         "Limit": 100
      },
      {
         "Endpoint": "*",
         "Period": "12h",
         "Limit": 1000
      },
      {
         "Endpoint": "*",
         "Period": "7d",
         "Limit": 10000
      }
   ]
},
```

For client or endpoint override steps follow this [link]( https://github.com/stefanprodan/AspNetCoreRateLimit/wiki/ClientRateLimitMiddleware#setup).

