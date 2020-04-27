# API - Throttling / Rate Limiting for ASP.NET Core

API throttling can easily be added to ASP.Net and ASP.Net Core using existing middleware NuGet packages.  Throttling allows you to control the rate of requests that a client can make to the API based on IP address, client API key, and request route.  Throttling can limit the number of calls allowed in a given period of time to either individual request routes, or across the whole API.  Specific IP address or client keys can optionally be whitelisted so they will not have throttling policies applied.  Similarly, specific IP address or client keys can optionally apply custom policies overriding global policies.  This document takes that one step further by adding a custom policy that identifies each client by their HTTP `Authorization` header, as well as allowing separate defaults for both basic and bearer authentications.  This is useful since bearer usually represents API to API requests, and bearer represents direct user to API requests.

## ASP.NET Core Throttling Steps

1. Add NuGet package [AspNetCoreRateLimit](https://github.com/stefanprodan/WebApiThrottle)

* If using ASP.NET Full Framework, use the following guide [API - Throttling / Rate Limiting for ASP.NET Full Framework](https://github.com/PaulGilchrist/documents/blob/master/articles/api-throttling-rate-limiting-for-asp-net-full-framework.md)

2. Add a custom rate limit configuration class that supports clients being identified using the HTTP `Authorization` header.

```cs
using AspNetCoreRateLimit;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Options;
using System;
using System.Collections.Generic;
using System.Linq;

namespace API.Classes {
    public class CustomRateLimitConfiguration : IRateLimitConfiguration {
        protected readonly IpRateLimitOptions IpRateLimitOptions;
        protected readonly ClientRateLimitOptions ClientRateLimitOptions;
        protected readonly ClientRateLimitPolicies ClientRateLimitPolicies;
        protected readonly IHttpContextAccessor HttpContextAccessor;
        protected readonly IClientPolicyStore ClientPolicyStore;
        public IList<IClientResolveContributor> ClientResolvers { get; } = new List<IClientResolveContributor>();
        public IList<IIpResolveContributor> IpResolvers { get; } = new List<IIpResolveContributor>();
        public virtual ICounterKeyBuilder EndpointCounterKeyBuilder { get; } = new PathCounterKeyBuilder();
        public virtual Func<double> RateIncrementer { get; } = () => 1;

        public CustomRateLimitConfiguration(IHttpContextAccessor httpContextAccessor, IClientPolicyStore clientPolicyStore, IOptions<IpRateLimitOptions> ipOptions, IOptions<ClientRateLimitOptions> clientOptions, IOptions<ClientRateLimitPolicies> clientPolicies) {            IpRateLimitOptions = ipOptions?.Value;
            ClientRateLimitOptions = clientOptions?.Value;
            ClientRateLimitPolicies = clientPolicies?.Value;
            HttpContextAccessor = httpContextAccessor;
            ClientPolicyStore = clientPolicyStore;
            ClientResolvers = new List<IClientResolveContributor>();
            IpResolvers = new List<IIpResolveContributor>();
            RegisterResolvers();
        }

        protected virtual void RegisterResolvers() {
            ClientResolvers.Add(new AuthTypeResolveContributor(HttpContextAccessor, ClientPolicyStore, ClientRateLimitOptions, ClientRateLimitPolicies));
        }
    }

    public class AuthTypeResolveContributor : IClientResolveContributor {
        private readonly ClientRateLimitOptions _clientOptions;
        private readonly ClientRateLimitPolicies _clientPolicies;
        private readonly IHttpContextAccessor _httpContextAccessor;
        private readonly IClientPolicyStore _clientPolicyStore;

        public AuthTypeResolveContributor(IHttpContextAccessor httpContextAccessor, IClientPolicyStore clientPolicyStore, ClientRateLimitOptions clientOptions, ClientRateLimitPolicies clientPolicies) {
            _clientOptions = clientOptions;
            _clientPolicies = clientPolicies;
            _httpContextAccessor = httpContextAccessor;
            _clientPolicyStore = clientPolicyStore;
        }

        string IClientResolveContributor.ResolveClient() {
            // General rules will be the default for individual users (bearer tokens), and the "basic" client rules will be for applications (basic tokens)
            if (!_clientOptions.ClientIdHeader.Equals("Authorization", StringComparison.OrdinalIgnoreCase) && _httpContextAccessor.HttpContext.Request.Headers.TryGetValue(_clientOptions.ClientIdHeader, out var headerValues)) {
                // The requestor is using a header other than "Authorization" so just return the clientId unchanged
                return headerValues.First();
            } else if (_httpContextAccessor.HttpContext.Request.Headers.TryGetValue("Authorization", out var authValues)) {
                var clientId = authValues.First();
                if (_clientPolicies.ClientRules.FirstOrDefault(r => r.ClientId.Equals(clientId, StringComparison.OrdinalIgnoreCase)) != null) {
                    // There is a specific client policy rule already set for for this "Authorization" client so just return the clientId unchanged
                } else {
                    if (clientId.StartsWith("basic", StringComparison.OrdinalIgnoreCase)) {
                        var rule = _clientPolicies.ClientRules.FirstOrDefault(r => r.ClientId.Equals("basic", StringComparison.OrdinalIgnoreCase));
                        if (rule != null) {
                            // Set this application to have the same rules as the default application rules 
                            _clientPolicyStore.SetAsync($"{_clientOptions.ClientPolicyPrefix}_{clientId}", new ClientRateLimitPolicy { ClientId = clientId, Rules = rule.Rules }).ConfigureAwait(false);
                        }
                    }
                }
                return clientId;
            } else {
                // No header was found so the user is anonymous and we will use the general rules
                return "anon";
            }
        }

    }

}
```

3. In the file `Startup.cs` and function `ConfigureServices()` add the following code before `services.AddMvc()`

```cs
// needed to load configuration from appsettings.json
services.AddOptions();
// needed to store rate limit counters and ip rules
services.AddMemoryCache();
// load general configuration from appsettings.json
services.Configure<ClientRateLimitOptions>(Configuration.GetSection("ClientRateLimiting"));
// load client rules from appsettings.json
services.Configure<ClientRateLimitPolicies>(Configuration.GetSection("ClientRateLimitPolicies"));
// inject counter and rules stores
services.AddSingleton<IClientPolicyStore, MemoryCacheClientPolicyStore>();
services.AddSingleton<IRateLimitCounterStore, MemoryCacheRateLimitCounterStore>();
```

4. In the file `Startup.cs` and function `ConfigureServices()` add the following code after `services.AddMvc()`

```cs
// configuration (resolvers, counter key builders)
services.AddSingleton<IRateLimitConfiguration, CustomRateLimitConfiguration>();
```

5. In the file `Startup.cs` and function `Configure()` add the following code before `app.UseMvc()`

```cs
app.UseClientRateLimiting();
```

6. Add default throttling settings to appsettings.json adjusting the below example to fit your requirements:

```json
"ClientRateLimiting": {
    "EnableEndpointRateLimiting": false,
    "StackBlockedRequests": false,
    "ClientIdHeader": "X-ClientId",
    "HttpStatusCode": 429,
    "EndpointWhitelist": [],
    "ClientWhitelist": [],
    "GeneralRules": [
        {
            "Endpoint": "*",
            "Period": "1s",
            "Limit": 5
        },
        {
            "Endpoint": "*",
            "Period": "1m",
            "Limit": 150
        },
        {
            "Endpoint": "*",
            "Period": "1h",
            "Limit": 4500
        },
        {
            "Endpoint": "*",
            "Period": "1d",
            "Limit": 54000
        }
    ]
},
"ClientRateLimitPolicies": {
    "ClientRules": [
        {
            "ClientId": "basic",
            "Rules": [
                {
                    "Endpoint": "*",
                    "Period": "1s",
                    "Limit": 100
                }
            ]
        },
        {
            "ClientId": "basic SpecificApp",
            "Rules": [
                {
                    "Endpoint": "*",
                    "Period": "1s",
                    "Limit": 200
                }
            ]
        }
    ]
},
```

7. Modify the file `Program.cs` to support loading the ClientPolicyStore using the following code.

```cs
using AspNetCoreRateLimit;
using Microsoft.AspNetCore;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.DependencyInjection;
using System.Threading.Tasks;

namespace ODataCoreTemplate {
    public static class Program {
        public static async Task Main(string[] args) {
            IWebHost webHost = CreateWebHostBuilder(args).Build();
            using (var scope = webHost.Services.CreateScope()) {
                // Get the ClientPolicyStore instance
                var clientPolicyStore = scope.ServiceProvider.GetRequiredService<IClientPolicyStore>();
                // Seed client data from appsettings
                await clientPolicyStore.SeedAsync();
            }
            await webHost.RunAsync();
        }

        public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
            WebHost.CreateDefaultBuilder(args)
                .UseApplicationInsights()
                .UseStartup<Startup>().UseKestrel(options => {
                    options.Limits.MaxRequestLineSize = 65536;
                });
    }
}
```
