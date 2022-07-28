# API Gateway Support (api, ingress, or proxy)

Gateways and proxies placed in front of your API will use layer 7 routing to redirect URL requests to specific backend services.  This process will re-write the original URL into a shorter service specific URL.  The Open API (Swagger) specification will be unaware of this URL change, and break Swagger UI features like `Try it now`.

Fixing this issue is fairly straight forward, starting with setting an environment variable that the API can use to identify what path the gateway or proxy will use to identify the service.  This would them be added to the Open API specification allowing the `try it now` feature to re-create the correct full URL rather than just assuming the API is at the root:

```cs
app.UseSwagger(options => {
    options.PreSerializeFilters.Add((swaggerDoc,httpReq) => {
        swaggerDoc.Servers = new List<OpenApiServer> { new OpenApiServer { Url = $"https://{httpReq.Host.Value}{Environment.GetEnvironmentVariable("BASE_PATH")}" } };
        // ...
    });
    // ...
});
```

Set the environment variable `BASE_PATH` to `/` when not behind a gateway or proxy.  This code can be found in the GitHub project named [saga-api](https://github.com/PaulGilchrist/saga-api).