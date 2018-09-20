Quote: Abraham Maslow
> <i>"If all you have is a hammer, everything looks like a nail"</i>

# Training Priorities

This document lists skills both required and useful for the different developer roles.  Skills identified as common apply to all roles.  Within each role, the skill lists are presented in priority order.

## Required Skills

### Required Skills - Common Developer

* Visual Studio 2017+ (or VS Code)
* JSON
* Interfaces/Repositories
* Dependency Injection
* Unit Testing
* Caching
* Azure Repos (previously in VSTS)
* Git (new skill that we should start training on today)
* OAuth Implicit Flow Bearer Tokens
* Application Insights
* Azure Boards (previously in VSTS)

### Required Skills - Front-End Developer

* HTML5
* CSS3
* JavaScript
* Bootstrap 4 (new skill that we should start training on today)
* Angular
* TypeScript
* Node Package Manager (NPM)
* SASS
* Jasmine/Karma
* Active Directory Authentication Library (ADAL)
* WebPack (team leads mostly)
* Pollyfills

### Required Skills - Back-End Developer

* C#
* .Net Core 2.1+
* Entity Framework Core 2.1+
* REST API
* Open API/Swagger & Swashbuckle
* OData
* LINQ
* API Versioning

### Required Skills - DevOps

* Azure PaaS
* Azure App Service (web app)
* Azure Storage
* Azure Pipelines (previously in VSTS)
* Azure Test Plans (previously in VSTS)
* Azure Artifacts (previously in VSTS)

### Required Skills - Database Developer

* T-SQL
* Azure SQL Database

## Additional Useful Skills

### Useful Skills - Common Developer

* VS Code
  * VS Code is Microsoft's cross OS, light weight, integrated development environment, and an alternative to Visual Studio.  Training should be optional, but many people feel you can be much more productive in VS Code than Visual Studio, so many developers would want to know both.
* Zero Downtime Deployments
  * A team should hold off on this training until they are ready to attempt zero downtime deployments for one of their applications.  This is a skill that will grow with experience, and mainly suited for applications like public websites or data integrations where downtime needs to be minimized.
* Code Scanning Static and Dynamic, Active and Passive, OWASP top 10, SANS top 25, and remediation
  * Would need to pick a solution before training could start.
* Containers
  * This technology would be something to consider for applications moving to the cloud that may not be a candidate for PaaS, or if wanting a multi-cloud solution to avoid vendor lock in.  Otherwise, a developer or DevOps person  only needs to have a very high level understanding of what it is and the problems it solves.
* Node JS
  * This is the engine that runs NPM and also the engine that runs Angular testing, code watching, re-compile, and browser refresh.  Node is also a server engine that runs on Windows, Linux, and Mac, making it more portable than IIS or other server engines.  Node can do much more than we use it for, and it would be valuable for developers to have a high level understanding of what it is, and its value proposition.  

### Useful Skills - Front-End Developer

* Web Workers
  * This technology could add value to almost any project so a solid understanding is recommended
* WebSockets
  * This technology could add value to almost any project so a solid understanding is recommended.  SignalR (server) and WebSocket (client) would most likely be trained on together.
* SVG & D3
  * Very project specific - Should have high level training only unless immediately applying to a current project
* Web Speech API
  * Very project specific - Should have high level training only unless immediately applying to a current project
* HTML Media Capture
  * Very project specific - Should have high level training only unless immediately applying to a current project

### Useful Skills - Back-End Developer

* Azure Event Grid & Webhooks
  * These technologies could add value to almost any project so a solid understanding is recommended.
* Graph Database
  * This technology could add value to almost any project so a solid understanding is recommended
* SignalR
  * This technology could add value to almost any project so a solid understanding is recommended.  SignalR (server) and WebSocket (client) would most likely be trained on together.
* Serverless Computing
  * These technologies could add value to almost any project so a solid understanding is recommended.
* Microservices
  * Very project specific - Should have high level training only unless immediately applying to a current project
* Blockchain
  * Very project specific - Should have high level training only unless immediately applying to a current project.  Most likely the industry will drive our need for Blockchain vs being driven internally.  This may still be a few years away before anything beyond a high level introduction is needed.
* Azure Immutable Storage
  * Very project specific - Should have high level training only unless immediately applying to a current project

### Useful Skills - DevOps

* Serverless Computing (common with back-end developer)
  * These technologies could add value to almost any project so a solid understanding is recommended.
* Containers (common with back-end developer)
  * This technology would be something to consider for applications moving to the cloud that may not be a candidate for PaaS, or if wanting a multi-cloud solution to avoid vendor lock in.  Otherwise, a developer or DevOps person  only needs to have a very high level understanding of what it is and the problems it solves.

### Useful Skills - Database Developer

* Azure Data Factory
  * Required - Will be used in the cloud similar to how SSIS is used on-prem today
* Azure Data Sync
  * Required - Microsoft's recommended method for Azure to Azure data sync (not HA/DR replication)
* In-Memory Database
  * This technology could add value to almost any project so a solid understanding is recommended.
* Column Store Indexes
  * This technology could add value to almost any project so a solid understanding is recommended.
* Temporal Tables
  * This technology could add value to almost any project so a solid understanding is recommended.
* SVG & D3 (For BI developer.  Common with front-end developer)
  * Very project specific - Should have high level training only unless immediately applying to a current project
