# API Versioning Usage

## Introduction

Versioning enables changes to be introduced into a production API, without those changes breaking existing clients.  When a new version is deployed, it runs alongside the existing version.  Clients can then move to the new version on their own timeline as long as they have fully moved over before the old version is removed (usually 6 months).

Versioning should be used for breaking changes only, which include things like removal of objects, actions, properties or relationships.  Versioning should not be used for non-breaking changes such as addition of objects, actions, properties or relationships unless those properties or relationships are required and have no default value.  Clients are expected to safely ignore any unknown (additive) properties and be resilient to the reordering of fields within the JSON objects.

APIs are publicly versioned into a Major.Minor numbering scheme. There may be an additional level to the versioning scheme (Major.Minor.Internal) used for bug fixes, security updates, and non-breaking changes.  The requested Major.Minor version should be part of the querystring as in the following example:

https://api.domain.com/odata/divisions?api-version=1.0

If omitted, the API will default to version 1.0.  Once version 1.0 is no longer supported, the default will shift to the lowest supported version which may already be deprecated.  It is recommended that the version be specified in the querystring to explicitly identify the version being requested.  

## Supported API Versions

An actively supported production API will usually have accessible 3 or more simultaneous versions.  Only one of these versions would be considered the “current” production version and should be used by the majority of clients.  Development on this version would be limited to bug fixes, security updates, and non-breaking changes.  The next version would be considered a “preview” version where breaking changes are developed, and clients can test them before they become generally available.  A preview version should not be used for production, and breaking changes can be added at any time up until it is released for general availability as the next current version.  At that time, the old current version would be considered a now “deprecated” version.  There may be more than one deprecated version, but for supportability and future progress reasons, it is best to keep them to a minimum (1 or 2).  Deprecated versions are fully supported but will become unsupported at some point in the not-too-distant future (usually 6 months or less).  It is best that clients move from deprecated versions to the current version at their earliest convenience.  Deprecated versions may receive bug fixes and security patches as needed but are frozen with respect to other non-breaking changes and all breaking changes.

Swagger maintains documentation on all supported versions and allows a client to select which version to view or test.  Swagger API descriptions will identify all deprecated and preview versions as well as recommend the current version to be used for production.  Swagger will also make it easier to identify deprecated functionality by greying out those HTTP actions while still keeping them fully documented and testable.  Swagger should also identify the date at which a deprecated version will be removed, once that date is known, or Swagger should identify it as an estimate only when not known.

In addition to Swagger, both the currently supported and deprecated versions will be separately identified in the header of all HTTP responses.

## Deprecation and Monitoring

Microsoft Application Insights tracks information such as the API version being accessed, the controller and action requested, and who the requestor is.  This information is used to identify the rate at which clients are moving away from deprecated versions, and who to contact before a deprecated version gets removed.  Deprecated versions must be removed in an acceptable time period or risk limiting future capabilities due to the need for SQL to support all current versions simultaniously.

## Limitations

All API versions deprecated, current, and preview, must interact with the same back end data source.  This places limitations on what can and cannot be achieved through versioning.  Usually there will need to be a single schema for the SQL tables, SQL views, SQL procedures, Entity Framework (EF) DBSets, EF Entities, and C# object models.  In some scenarios, EF Core 2.0 table splitting can be used to allow two C# object models pointing to two Entity Framework DBSets both pointing to the same SQL table.  This should be avoided as much as possible due to the complexity and potential for conflicts.  Using this design, both models still need to enforce the single SQL table's data types, nullability, and relationship requirements.

The controller code with their available HTTP actions, OData models and EntitySet definitions, are easier to change between versions.  These can be used to ignore C# object models and Entity Framework DBSet entity properties.  For example, removing a property like middle name on a contact object would be a breaking change.  The SQL database table would still contain the middle name property, as would EF DBSet and C# object model, but the OData entity would hide it from the current API version while still exposing it to the deprecated versions.  Similar and potentially more complex things can be done within the HTTP action code as controllers should be separated across versions.

## Implementing within the API

Details on adding versioning support to an existing API .Net core project can be found here.  Once versioning is supported within the project, the following steps should be used when implementing a new “preview” version within the API.  

1.	Create new version folder under the controller’s folder using the format VMajor.Minor where Minor can be ignored if 0.  Examples include V1, V1.1, V1.2, V2, V2.1, etc.
2.	Copy all controllers from current version’s folder to preview version’s folder
3.	Search and replace within this new folder, only changing the old version’s name to new version’s name, and old namespace to new namespace.  Both the namespace and version should be at the top of the controller just before the controller’s class definition.  It is the namespace that allows simultaneous controllers of the same name to exist within the project.
4.	Update Swagger documentation of which version is current, preview, and deprecated in the file ConfigureSwaggerOptions.cs and function CreateInfoForApiVersion using the following code updated with the current version number:
            if (description.ApiVersion.MajorVersion<2 ) {
                info.Description = "This API version has been deprecated.  Please use version V2";
            }
            if (description.ApiVersion.MajorVersion > 2) {
                info.Description = "This API version is in preview.  Please use version V2 for production applications";
            }
5.	Modify OData model configurations and/or API controllers as needed to enable new breaking change functionality.  Bug fixes, or security patches should be implemented in all versions, and non-breaking changes should be implemented in both the “preview” version and “current” version only.

## References
* [Microsoft REST API Guidelines](https://github.com/Microsoft/api-guidelines?WT.mc_id=-blog-scottha)
* [https://github.com/microsoft/aspnet-api-versioning/wiki](https://github.com/microsoft/aspnet-api-versioning/wiki)
* [Entity Framework 2.0 Table Splitting](https://docs.microsoft.com/en-us/ef/core/what-is-new/ef-core-2.0)
* [Table Splitting/Table Sharing](https://docs.microsoft.com/en-us/ef/core/modeling/table-splitting)
* [Versioning Setup for OData on ASP.Net  Core](https://pultevsts.visualstudio.com/IT/_git/DeveloperCommunityDocumentLibrary?path=%2Farticles%2Fapi%2Fapi-odata-versioning-setup-for-asp-net-core.md&version=GBmaster)

