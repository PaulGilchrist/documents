# API - Adding Azure Application Insights

Azure Application Insights allows for collecting, reporting, and alerting on both server and client side telemetry.  Common server side telemetry includes requests, events, exceptions, and trace data.  The following steps will add the ability for a new or existing .Net Core application to send Application Insights telemetry to Azure and include information from the HTTP context such as user identity:

1. Add the NuGet package named `Microsoft.ApplicationInsights.AspNetCore`

2. Create a new class using the below code:
   * This class will be used to insert user information into the telemetry data.

```c#
using Microsoft.ApplicationInsights.Channel;
using Microsoft.ApplicationInsights.DataContracts;
using Microsoft.ApplicationInsights.Extensibility;
using Microsoft.AspNetCore.Http;

namespace API.Classes {
    public class TelemetryInitializer : ITelemetryInitializer {
        IHttpContextAccessor httpContextAccessor;

        public TelemetryInitializer(IHttpContextAccessor httpContextAccessor) {
            this.httpContextAccessor = httpContextAccessor;
        }

        public void Initialize(ITelemetry telemetry) {
            var requestTelemetry = telemetry as RequestTelemetry;
            if (httpContextAccessor.HttpContext != null && httpContextAccessor.HttpContext.User.Identity.Name != null) {
                requestTelemetry.Context.User.Id = httpContextAccessor.HttpContext.User.Identity.Name;
            }
        }
    }
}
```

3. Make sure the following code is added below `AddMvc` and within the `ConfigureServices()` function of the `Startup.cs` file

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
