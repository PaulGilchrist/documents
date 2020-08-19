# API - Adding Azure Application Insights to ASP.Net Core

Azure Application Insights allows for collecting, reporting, and alerting on both server and client side telemetry.  Common server side telemetry includes requests, events, exceptions, and trace data.  The following steps will add the ability for a new or existing .Net Core application to send Application Insights telemetry to Azure and include information from the HTTP context such as user identity:

1. Add the NuGet package named `Microsoft.ApplicationInsights.AspNetCore`

2. Create a new class named `TelemetryInitializer` using the below code:
   * This class will be used to insert user information into the telemetry data.

```c#
// https://github.com/Microsoft/ApplicationInsights-aspnetcore/issues/562

using Microsoft.ApplicationInsights.Channel;
using Microsoft.ApplicationInsights.DataContracts;
using Microsoft.ApplicationInsights.Extensibility;
using Microsoft.AspNetCore.Http;

namespace API.Classes {
    public class TelemetryInitializer : ITelemetryInitializer {
        IHttpContextAccessor _httpContextAccessor;

        public TelemetryInitializer(IHttpContextAccessor httpContextAccessor) {
            _httpContextAccessor = httpContextAccessor;
        }

        public void Initialize(ITelemetry telemetry) {
            RequestTelemetry requestTelemetry = telemetry as RequestTelemetry;
            if (requestTelemetry != null && _httpContextAccessor.HttpContext != null) {
                if (_httpContextAccessor.HttpContext.User.Identity.Name != null) {
                    requestTelemetry.Context.User.Id = _httpContextAccessor.HttpContext.User.Identity.Name;
                    requestTelemetry.Context.User.AuthenticatedUserId = _httpContextAccessor.HttpContext.User.Identity.Name;
                }
                if (_httpContextAccessor.HttpContext.Items.ContainsKey("RequestBody")) {
                    requestTelemetry.Properties.Add("body", (string)_httpContextAccessor.HttpContext.Items["RequestBody"]);
                }
            }
        }
    }
}
```

3. Add the following code above `AddMvc()` and within the `ConfigureServices()` function of the `Startup.cs` file

```cs
    // The following line enables Application Insights telemetry collection.
    services.AddApplicationInsightsTelemetry();
```

3. Add the following code below `AddMvc()` and within the `ConfigureServices()` function of the `Startup.cs` file

```c#
services.AddHttpContextAccessor();
```

4. Pass in `IHttpContextAccessor httpContextAccessor` to the `Configure()` function of the `Startup.cs` file

5. Make sure the following code is added above `UseMvc` and within the `Configure()` function of the `Startup.cs` file

```c#
// Add custom telemetry initializer to add user name from the HTTP context
var configuration = app.ApplicationServices.GetService<TelemetryConfiguration>();
configuration.TelemetryInitializers.Add(new TelemetryInitializer(httpContextAccessor));
```
