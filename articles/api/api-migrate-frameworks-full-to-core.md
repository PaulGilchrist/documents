# Converting an Object from ASP.Net Full Framework to Core Framework

This document will walk through the steps needed to duplicate one API object and controller in an ASP.Net Full Framework project over to an ASP.Net Core project.  This document assumes you have already built the foundation for the ASP.Net Core project to match the capabilities of the existing Full Framework project (such as OData, Swagger, OAuth, etc.).  See separate documentation within this library, related to ASP.Net Core setup of these foundational capabilities.

After going through this process a few times, it should take less than an hour for the average controller, but you should allow more time initially when learning this process, and the project differences as a whole.

## Steps

1) Create model matching full framework
   * Remove all `System.Diagnostics.CodeAnalysis.SuppressMessages`
   * Although both work, change Nullable<T> to <T>? to better fit with modern .Net Core documentation and example reference materials
   * If the object contains any enumerations that are not already defined in their own class files, add those `enum` classes at this time
   * Comment out related collections that have not yet been defined, and their associated hash sets in the constructor - Remember to uncomment them as their related models get defined (step #7 below)
   * At the time of this writing, many to many relationships in core do not support hidding/skipping the association table.  This means model changes may be neccessary changing the relationship to the association table rather than directly to the other model (breaking change to API contract)
2) Add in validation annotations as defined in the full framework project's respective model validation class
3) Update any related existing models (if any) by uncommenting their references to this model
4) Add `DbSet<T>`
   * May also add any related DbSets at this time
5) Add context `Entity<T>`
   * May also add any related context entities at this time, or update existing context entities that relate to the new entity
   * Map context entity to database table
     * ex: `entity.ToTable("Division", schema: "org");`
   * If single primary key, identify it
     * ex: `entity.HasKey(e => e => e.Id);`
   * If a composite key, define it's composition
     * ex: `entity.HasKey(e => new { e.DivisionId, e.Tag });`
   * Add any differences between database column names and model property names
     * ex: `entity.Property(x => x.Id).HasColumnName("DivisionId");`
   * Add any conversions needed to transform from Enum properties to SQL type
     * ex: `entity.Property(c => c.Status).HasConversion<byte>().HasColumnName("StatusId");`
   * Add any relationships to other entities recommended to always go from one to many
     * ex: `entity.HasOne(e => e.Area).WithMany(e => e.Divisions).HasForeignKey(e => e.AreaId);`
6) Add OData EntitySet<T>
   * May also add any related OData entitySets at this time (ex: many to many tables)
   * ex: `builder.EntitySet<Division>("divisions").EntityType.Count().Expand(5).Filter().OrderBy().Page().Select();`
   * If EntitySet has a compositie key, define it's composition
     * ex: `builder.EntitySet<DivisionTag>("divisionTags").EntityType.HasKey(e => new { e.DivisionId, e.Tag }).Count().Expand(5).Filter().OrderBy().Page().Select();`
7) Add Controller
   * It may be quickest to start by copying an existing similar ASP.Net core controller, then modifying it to match any custom functionality and security implemented in the Full Framework controller
   * Check for error codes being returned that are not in the Swagger attributes
     * Ex: POST may return `Conflict` 409 but be missing from attributes
8) Update related controllers' REF functions (if any)
9) Unit test controller actions making sure to also test expanding into related objects
   * Also unit test related objects expanding back into the new object

## References
  * https://docs.microsoft.com/en-us/ef/core/get-started/aspnetcore/new-db
