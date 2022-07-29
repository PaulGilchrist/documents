# API Health Check Endpoints for Liveness and Readiness Probes

Microsoft supports API healthcheck endpoints through the `Microsoft.AspNetCore.Diagnostics.HealthChecks` namespace.  These endpoints can be used for monitoring the health of the API as well as backend dependency services.  They can also be used by [Kubernetes liveness and readiness probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).

These probes in turn can determine when to begin sending traffic to a new container, or when that container is unhealthy and should be replaced.  This is ultimately better than how Azure App Service defines when an API is ready as you can control what defines your service and prevent failed connections.

The first step in setting up a health check is determining what you want in the readiness response.  Here is a good starting example:

```cs
static Task WriteReadinessResponse(HttpContext context, HealthReport result) {
    context.Response.ContentType = "application/json; charset=utf-8";
    var options = new JsonWriterOptions {
        Indented = true
    };
    using (var stream = new MemoryStream()) {
        using (var writer = new Utf8JsonWriter(stream, options)) {
            writer.WriteStartObject();
            writer.WriteString("status", result.Status.ToString());
            writer.WriteStartObject("results");
            foreach (var entry in result.Entries) {
                writer.WriteStartObject(entry.Key);
                writer.WriteString("status", entry.Value.Status.ToString());
                writer.WriteString("description", entry.Value.Description);
                writer.WriteStartObject("data");
                foreach (var item in entry.Value.Data) {
                    writer.WritePropertyName(item.Key);
                    JsonSerializer.Serialize(
                        writer, item.Value, item.Value?.GetType() ??
                        typeof(object));
                }
                writer.WriteEndObject();
                writer.WriteEndObject();
            }
            writer.WriteEndObject();
            writer.WriteEndObject();
        }
        var json = Encoding.UTF8.GetString(stream.ToArray());
        return context.Response.WriteAsync(json);
    }
}
```

Next, from within `Program.cs` add the default health checks

```cs
var healthCheckBuilder = builder.Services.AddHealthChecks();
```

And any optional dependency checks (add NuGet packages as needed)

```cs
healthCheckBuilder.AddAzureServiceBusTopic(applicationSettings.QueueConnectionString, "contacts", null, Microsoft.Extensions.Diagnostics.HealthChecks.HealthStatus.Unhealthy);

healthCheckBuilder.AddRabbitMQ(uri, null, null, Microsoft.Extensions.Diagnostics.HealthChecks.HealthStatus.Unhealthy);

healthCheckBuilder.AddMongoDb(applicationSettings.DatabaseConnectionString, applicationSettings.DatabaseName, null, Microsoft.Extensions.Diagnostics.HealthChecks.HealthStatus.Unhealthy);
```

And finally, map the API endpoint path or paths

```cs
app.MapHealthChecks("/health/readiness", new HealthCheckOptions() { ResponseWriter = WriteReadinessResponse }).AllowAnonymous();

app.MapHealthChecks("/health/liveness", new HealthCheckOptions() { Predicate = (_) => false }).AllowAnonymous();
```