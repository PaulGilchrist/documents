# API - Adding Azure Application Insights to ASP.Net Full Framework

Azure Application Insights allows for collecting, reporting, and alerting on both server and client side telemetry.  Common server side telemetry includes requests, events, exceptions, and trace data.  The following steps will add the ability for a new or existing .Net Core application to send Application Insights telemetry to Azure and include information from the HTTP context such as user identity:

1. Add the NuGet package named `Microsoft.ApplicationInsights`

2. Create a new class using the below code:
   * This class will be used to insert user information into the telemetry data.

```c#
    public class TelemetryInitializer : ITelemetryInitializer {
        public void Initialize(ITelemetry telemetry) {
            var httpContext = HttpContext.Current;
            // If called as part of request execution and not from some async thread
            if (httpContext!=null && httpContext.User!=null) {
                telemetry.Context.User.Id = httpContext.User.Identity.Name;
            }
        }
    }
```

3. Add the following line to File `ApplicationInsights.config` in section `<TelemetryInitializers>`.  Adjust the application name of `API` and class namespace of `API.Classes` accordingly.

```xml
<Add Type="API.Classes.TelemetryInitializer, API"/>
```
