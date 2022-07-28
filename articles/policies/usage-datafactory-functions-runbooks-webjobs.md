# Recommendations when choosing between Azure Data Factory, Azure Functions, Azure Runbooks, and Web Jobs

Many PaaS and SaaS services offered in Microsoft Azure overlap in their capabilities making it sometimes difficult to determine which should be used in different scenarios.  This document is a quick list of their strength and weaknesses to help with making the right choice.  The following list is in alphabetical (not priority) order.

## Azure Data Factory
* Does not require knowing C#, so best when development and support will be managed by database developers rather than full stack developers
* Good for data intensive, batch operations with non-complex transformations
* Has ability to run existing SSIS packages
* Has a native SFTP connector, as well as other useful connectors
* Has native OData connector, but currently does not support $expand
* Has native REST connector which does support OData including $expand, but currently does not support the `Lookup` action
## Azure Functions
* Most common developer skillset making for ease of maintainability
* Most flexible triggering mechanisms
* Most standardized source control workflow
* Best for complex processing or data enrichment
* Best for portability
* Best for code re-use and modularity
* Best integration with current logging, monitoring, and alerting standards through seamless integration of Application Insights
## Runbooks
* Does not require knowing C#, so best when development and support will be managed by operations using PowerShell rather than full stack developers
## Web Jobs
* Better for long running processes
* Shares App Service Plan with website or API
