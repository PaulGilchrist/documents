# High Availability and Scale using HttpClientFactory and Polly

This document will discuss how to use the HttpClient pooling capabilities of `HttpClientFactory`, and the retry, backoff, and circuit breaker capabilities of `Polly` for increased availability and scale for any .Net API project.

Add the `Microsoft.Extensions.Http.Polly` NuGet package to the project then add the following functions accessable from `Program.cs`, tuning or adding to these functions as desired.

```cs
using Polly;
using Polly.Extensions.Http;

static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy() {
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(5, TimeSpan.FromSeconds(30));
}

static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy() {
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .OrResult(msg => msg.StatusCode == System.Net.HttpStatusCode.NotFound)
        .WaitAndRetryAsync(6, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));
}
```

Next add the policies when creating the HttpClientFactory in the same `Program.cs` file:

```cs
builder.Services.AddHttpClient("DefaultHttpClient")
    .SetHandlerLifetime(TimeSpan.FromMinutes(5))  //Set lifetime to five minutes
    .AddPolicyHandler(GetRetryPolicy())
    .AddPolicyHandler(GetCircuitBreakerPolicy());
```

The above example code will allow HttpClient to be re-used from the HttpCleintFactory for up to 5 minutes, handle transient failures automatically, retry any failed requests up to 6 times with an exponential backoff, and go into cuircuit breaker mode for 30 seconds should the retries fail 5 consecutive times.

This code can be found in the GitHub project named [saga-api](https://github.com/PaulGilchrist/saga-api).

