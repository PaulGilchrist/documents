# API Swagger/Open API for ASP.Net Core using NSWag

1. Install NuGet package named `NSWag.AspNetCore`
2. In the file `Startup.cs` function `Configure()`, add the following new lines, changing the title and description appropriately

   ```cs
   app.UseSwaggerUi(typeof(Startup).GetTypeInfo().Assembly, settings => {
      settings.GeneratorSettings.Title="ODataCoreTemplate";
      settings.GeneratorSettings.Description="OData .Net Core OpenAPI Project Template";
      settings.GeneratorSettings.IsAspNetCore = true;
      // settings.GeneratorSettings.SerializerSettings.
   });
   ```

3. To every controller HTTP Action, add a `/// <summary></summary>` comment that describes the object and action
4. To every controller HTTP Action, add a `/// <remarks>` multi-line comment that describes all security rules, business rules, and other important information not covered by antoehr more specific comment type
5. To every controller HTTP Action, add `/// <param name=”<name>”></param>` comments for each non-OData input parameter that describes the purpose and usage for the given input parameter
6. To every controller HTTP Action, add `/// <response code=”<code#>”></response>` comments for each possible response code.  This allows a developer using the API to know what possible responses to test for.

References
* [Video – NSWag Tutorial](https://www.youtube.com/watch?v=lF9ZZ8p2Ciw)
