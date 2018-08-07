# API - Throttling / Rate Limiting for ASP.NET Full Framework

API throttling can easily be added to ASP.Net and ASP.Net Core using existing middleware NuGet packages.  Throttling allows you to control the rate of requests that a client can make to the API based on IP address, client API key, and request route.  Throttling can limit the number of calls allowed in a given period of time to either individual request routes, or across the whole API.  Specific IP address or client keys can optionally be whitelisted so they will not have throttling policies applied.  Similarly, specific IP address or client keys can optionally apply custom policies overriding global policies.

## ASP.NET Full Framework Steps

1. Add NuGet package [WebApiThrottle](https://github.com/stefanprodan/WebApiThrottle)
   * If using .Net Core use package [AspNetCoreRateLimit](https://github.com/stefanprodan/AspNetCoreRateLimit)
2. Create new class file names CustomThrottlingHandler

```cs
public class CustomThrottlingHandler : ThrottlingHandler {
   protected override RequestIdentity SetIdentity(HttpRequestMessage request) {
      return new RequestIdentity() {
         ClientKey = request.Headers.Contains("Authorization") ? request.Headers.GetValues("Authorization").First() : "noToken",
         ClientIp = base.GetClientIp(request).ToString(),
         Endpoint = request.RequestUri.AbsolutePath.ToLowerInvariant()
      };
   }
}
```

3. In the file `WebApiConfig.cs` and function `Register`, add the below code:

```cs
// Throttling - https://github.com/stefanprodan/AspNetCoreRateLimit
config.MessageHandlers.Add(new CustomThrottlingHandler() {
   Policy = new ThrottlePolicy(
      perSecond: Convert.ToInt32(ConfigurationManager.AppSettings["throttle:limitPerSecond"]),
      perMinute: Convert.ToInt32(ConfigurationManager.AppSettings["throttle:limitPerMinute"]),
      perHour: Convert.ToInt32(ConfigurationManager.AppSettings["throttle:limitPerHour"]),
      perDay: Convert.ToInt32(ConfigurationManager.AppSettings["throttle:limitPerDay"]),
      perWeek: Convert.ToInt32(ConfigurationManager.AppSettings["throttle:limitPerWeek"])
   ) {
      IpThrottling = true,
      ClientThrottling = true,
      EndpointRules = new Dictionary<string, RateLimits> {
         "swagger", new RateLimits {
            PerSecond = long.MaxValue,
            PerMinute = long.MaxValue,
            PerHour = long.MaxValue,
            PerDay = long.MaxValue,
            PerWeek = long.MaxValue
         }
      }
   },
   Repository = new CacheRepository()
});
```

4. Add variables to web.config allowing later control from Azure settings

```cs
<add key="throttle:limitPerSecond" value="1" />
<add key="throttle:limitPerMinute" value="30" />
<add key="throttle:limitPerHour" value="900" />
<add key="throttle:limitPerDay" value="10800" />
<add key="throttle:limitPerWeek" value="37800" />
```

5. Test

For whitelisting or endpoint throttling steps follow this [link]( https://github.com/stefanprodan/WebApiThrottle#ip-andor-client-key-white-listing).
