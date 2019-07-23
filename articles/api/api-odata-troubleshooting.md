# OData Troubleshooting

This document will go step by step through troubleshooting and working around an example OData expand performance issues.  We will start with the below example OData query that looks simple enough on the surface.

```javascript
{{baseUrl}}/salesAgreements(616)?$expand=jobSalesAgreementAssocs($filter=isActive eq true;$select=id;$expand=job($select=id;$expand=lot($select=lotBlock),financialCommunity($select=id,number;$expand=market($select=number)))),buyers($select=id;$filter=isPrimaryBuyer eq true;$expand=opportunityContactAssoc($select=id;$expand=contact($select=firstName,lastName)))
```

The first step would be to make the query more readable to better understand what it is doing

```javascript
{{baseUrl}}//salesAgreements(616)?
   $expand=
      jobSalesAgreementAssocs(
         $select=id;
         $filter=isActive eq true;
         $expand=
            job(
               $select=id;
               $expand=
                  lot(
                     $select=lotBlock
                  )
                ,financialCommunity(
                   $select=id,number;
                   $expand=
                      market(
                         $select=number
                      )
                )
            )
      )
      ,buyers(
         $select=id;
         $filter=isPrimaryBuyer eq true;
         $expand=
            opportunityContactAssoc(
               $select=id;
               $expand=contact(
                  $select=firstName,lastName
               )
            )
      )
```

Looking at the query we see it returns 9 separate object nested 5 levels deep.  The next step would be to narrow down with level of the nesting is causing the most performance impact.  This can be done by breaking the query down into parts and timing each.  Using a tool like [PostMan](https://www.getpostman.com/), we start this by querying just the ```salesAgreement``` without expands and see it responds in less than 1 second.

```javascript
{{baseUrl}}//salesAgreements(616)
```

We follow this by querying the ```jobSalesAgreementAssocs``` along with all its sub-nested objects.  This test also returns in less than 1 second.

```javascript
{{baseUrl}}/jobSalesAgreementAssocs?$select=id&$filter=salesAgreementId eq 616 and isActive eq true&$expand=job($select=id;$expand=lot($select=lotBlock),financialCommunity($select=id,number;$expand=market($select=number)))
```

We do the sMe thing for ```buyer``` and find it takes a full 99 seconds to return.  In rare circumstances, this query may even timeout or result in an out of memory exception.

```javascript
{{baseUrl}}/salesAgreements(616)?$expand=buyers($select=id;$filter=isPrimaryBuyer eq true;$expand=opportunityContactAssoc($select=id;$expand=contact($select=firstName,lastName)))
```

This tells us we need to further break down the ```buyer``` query so we start by looking at the ```buyer``` object without its nested objects.  This test returns in less than 1 second so we know it must be one of the nested expands causing the performance issue.  

```javascript
{{baseUrl}}/salesAgreements(616)?$expand=buyers($select=id;$filter=isPrimaryBuyer eq true)
```

We next add in the ```opportunityContactAssoc``` and the timing remains under 1 second.  This indicates that the issue is with expanding of the ```contact``` object nested under ```opportunityContactAssoc```.

```javascript
{{baseUrl}}/salesAgreements(616)?$expand=buyers($select=id;$filter=isPrimaryBuyer eq true;$expand=opportunityContactAssoc($select=id))
```

Now that we have identified where the performance bottleneck is occuring within the query, we need to look at the TSQL that OData is generating.  This can be done by debugging the code within Visual Studio.  First make sure modify the ```appsettings.json``` file setting the logging level to ```Information``` so OData will log the TSQL being generated.

```json
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  },
```

Next, open the project properties and ensure the ```ASPNETCORE_ENVIRONMENT``` is pointing to the correct environment where the performance issue is occuring.  This will ensure we are connected to the correct backend database matching our ```PostMan``` tests.

Launch the debugger and wait for the application to fully load.  Next, open the ```Output```, window, and select ```Clear All```, immediatly followed by executing the simplest OData query that recreates the issue.  In our case it was the ```buyer``` query along with all its sub-nested objects.

```javascript
{{baseUrl}}/salesAgreements(616)?$expand=buyers($select=id;$filter=isPrimaryBuyer eq true;$expand=opportunityContactAssoc($select=id;$expand=contact($select=firstName,lastName)))
```

Looking at the ```Output``` window we see the following TSQL being generated from OData:

```SQL
SELECT [e].[SalesAgreementId], [e].[ApprovedDate], [e].[CreatedBy], [e].[CreatedUtcDate], [e].[ECOEDate], [e].[InsuranceQuoteOptIn], [e].[LastModifiedBy], [e].[LastModifiedUtcDate], [e].[LenderTypeId], [e].[PropertyTypeId], [e].[SalePrice], [e].[SalesAgreementNumber], [e].[SignedDate], [e].[StatusId], [e].[StatusUtcDate], [e].[TrustName]
FROM [sales].[SalesAgreement] AS [e]
WHERE [e].[SalesAgreementId] = 616

SELECT [t].[OpportunityContactAssocId] AS [Value], N'45f8c3f2-0809-4898-a418-c2ae45bcd65b' AS [ModelID], N'opportunityContactAssoc' AS [Name3], N'contact' AS [Name2], N'firstName' AS [Name1], N'lastName' AS [Name], N'id' AS [Name0], NULL, [t].[SalesAgreementBuyerId] AS [Value0], CASE
    WHEN [t].[OpportunityContactAssocId] IS NULL
    THEN CAST(1 AS BIT) ELSE CAST(0 AS BIT)
END AS [IsNull]
FROM (
    SELECT [s].*
    FROM [sales].[SalesAgreementBuyer] AS [s]
    WHERE 616 = [s].[SalesAgreementId]
) AS [t]
WHERE [t].[IsPrimaryBuyer] = 1

SELECT [$it.OpportunityContactAssoc.Contact].[ContactId], [$it.OpportunityContactAssoc.Contact].[FirstName], [$it.OpportunityContactAssoc.Contact].[LastName]
FROM [shr].[Contact] AS [$it.OpportunityContactAssoc.Contact]
```

Any of these above queries could now be executed using ```SQL Server Management Studio (SSMS)``` to see their individual run times.  They could also be checked for the proper column indexing to ensure it matches up with the table ```join``` and ```where``` clauses used.  In our case, the issue becomes immediatly evident so these additional steps are not neccessary.

The first query looks good and gets the ```salesAgreement``` object.  The second query gets both the ```buyer``` and the ```opportunityContactAssoc``` and seems to have the appropriate joins and where clause to run with good performance.  It is the last query that is the root of our issue.

```SQL
SELECT [$it.OpportunityContactAssoc.Contact].[ContactId], [$it.OpportunityContactAssoc.Contact].[FirstName], [$it.OpportunityContactAssoc.Contact].[LastName]
FROM [shr].[Contact] AS [$it.OpportunityContactAssoc.Contact]
```

This query does not properly limit the results to only the one contact identified in the ```OpportunityContactAssoc``` object but rather queries every contact in the database.  With the database containing millions of contacts, this can cause a tremendous performance hit or even an out of memroy condition.  It is unclear why OData writes the TSQL this way, but it may be related to the [N+1 Bug #1653](https://github.com/OData/WebApi/pull/1653) where OData thinks due to the nexting that it would be better to get all the data and later join it in memory rather than allowing SQL to do the join itself.

Until Microsoft or the OData development team correct this bug, we need to implement a workaround.  The workaround would be to execute the initial ```salesAgreement``` query without the nested ```contact``` object, and then separatly get the ```contact``` object and attach it to ```opportunityContactAssoc``` manually.

The below code would achieve that workaround:

```cs
```
