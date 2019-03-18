# Converting an Object from ASP.Net Full Framework to Core Framework

This document will walk through the steps needed to duplicate one API object & controller in an ASP.Net Full Framework project over to an ASP.Net Core project.  This document assumes you have already build the foundation for the ASP.Net Core project to match the capabilities of the existing Full Framework project (such as OData, Swagger, OAuth, etc.).  See separate documentation within this library, related to ASP.Net Core setup of these foundational capabilities.

After going through this process a few times, it should take less that an hour for the average controller, but you should allow more time initially when learning this process, and the project differences as a whole.

## Steps

1) Create model matching full framework
   * Remove class constructor and associated hash sets
   * Remove all `System.Diagnostics.CodeAnalysis.SuppressMessages`
   * If the object contains any enumerations that are not already defined in their own class file, add those `enum` classes at this time
   * Comment out related collections that have not yet been defined - Remember to uncomment them as their related models get defined
   * Add in validation annotations as defined in the full framework project's respective model validation class

2) Add `DbSet<T>`
3) Optional (not recommended) - Scaffold single table
   * Frequently easier to just look at table columns in `SQL Server Management Studio` when mapping existing EDH model to context entity.  Even when scaffolding, you need to first use a tool like SQL Server Management Studio to find the table name to scaffold
   * If scaffolding, make sure the output is to a dedicated `Scaffold` folder and not to the `Models` folder other than the models.  This prevents overwriting non-scaffolded models, and allows easy comparisons between the API's model, and database's scaffolded model.

```ps
Scaffold-DbContext "data source=<database-dns-name>;initial catalog=<database-name>;persist security info=True;user id=<username>;password=<password>;MultipleActiveResultSets=True;" Microsoft.EntityFrameworkCore.SqlServer -OutputDir Scaffold -Tables org.Division
```

4) Add `Entity<T>`
   * Map context entity to database table
     * ex: `Entity<Division>().ToTable("Division", schema: "org");`
   * Add any differences between database column names and model property names
     * ex: `divisions.Property(x => x.Id).HasColumnName("DivisionId");`
   * Add any conversions needed to transform from Enum properties to SQL type
     * ex: `divisions.Property(c => c.ConstructionStatusDescription).HasConversion<byte>().HasColumnName("FinancialStatusId");`

5) Add Controller

   * It may be quickest to start by copying an existing similar ASP.Net core controller, then modifying it to match any custom functionality and security implemented in the Full Framework controller
   * Check for error codes being returned that are not in the Swagger attributes
     * Ex: POST may return `Conflict` 409 but be missing from attributes

6) Add version configuration class for respective controller
7) Go back to any previously created objects that relate to this object, and uncomment the relationship within its model and entity definitions
8) Unit test controller actions making sure to also test expanding into related objects

## References
  * https://docs.microsoft.com/en-us/ef/core/get-started/aspnetcore/new-db
