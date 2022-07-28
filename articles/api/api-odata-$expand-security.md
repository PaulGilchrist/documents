# OData $Expand Security

## Issue
C# Web Api's `[Authorize]` attribute only has the ability to secure an API endpoint, and no additional granularity to secure individual objects or properties returned by that endpoint.  When combined with OData $expand, and user may access an API endpoint which they have permissions to, then attempt to $expand into an object which they do not have permissions to.  Without adding custom security, the user would be successful in their attempt to exceed their permissions in this way, and access data they should not see.

## Solution
Create a new attribute named "RestrictedEnableQueryAttribute" to replace the default OData `[EnableQuery]` attribute.  The purpose of this new attribute will be to validate every $expand within every query ensuring that the current user identity has permissions to every object being expanded into.  If validated, the query executes as normal.  If validation fails, the query is not executed, and instead, an exception is thrown identifying the user and object they were attempting to $expand into that was not allowed based on their currently assigned permissions.

## Code Example
```cs
using Microsoft.AspNetCore.OData.Query;
using Microsoft.OData;
using Microsoft.OData.UriParser;

public class RestrictedEnableQueryAttribute: EnableQueryAttribute {
    public RestrictedEnableQueryAttribute() : base() { }

    public override void ValidateQuery(HttpRequest request,ODataQueryOptions queryOpts) {
        if(queryOpts.SelectExpand != null) {
            var firstFailedExpandClause = queryOpts.SelectExpand.SelectExpandClause.SelectedItems.OfType<ExpandedNavigationSelectItem>().FirstOrDefault(o => {
                // Make sure that the object being expanded into is allowed based on the user's role claims
                // NavigationSource.Name will match OdataModel.EntitySet.Name
                return !request.HttpContext.User.HasClaim("http://schemas.microsoft.com/ws/2008/06/identity/claims/role", $"Get-{o.NavigationSource.Name}");
            }, null);
            if(firstFailedExpandClause != null) {
                throw new ODataException($"{request.HttpContext.User.Identity?.Name} lacks permissions to $expand into '{firstFailedExpandClause.NavigationSource.Name}'");
            }
        }
        base.ValidateQuery(request,queryOpts);
    }
}
```