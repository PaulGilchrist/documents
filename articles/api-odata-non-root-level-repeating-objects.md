# OData Non-Root Level Repeating Objects

Objects like `Tags` or `Notes` are examples of non-root level objects that exist only in the context of a parent object.  They may however have unique object definitions within the database, but still be represented in the API using the same name.  For example:  The database may have a `UserNotes` table and a completely different `AddressNotes` table.  We don't want to expose new API endpoints named `User({id})/UserNotes` and `Addresses({id})/AddressNotes` but rather repeat the object as just `Notes` made unique through its parent's paths `User({id})/Notes` and `Addresses({id})/Notes`.

This document will discuss how to setup OData EntitySets so they will properly support repeated object names.

1. Update the impacted object models with the repeaded object name.

```cs
public class Address {
    public List<AddressNote> Notes { get; set; } = new List<AddressNote>();
    ...
}

public class User {
    public List<UserNote> Notes { get; set; } = new List<UserNote>();
    ...
}
```

2. Update the class `ApiDbContext`, function `OnModelCreating` to reflect the property name change.

```cs
modelBuilder.Entity<Address>().HasMany(a => a.Notes);
modelBuilder.Entity<User>().HasMany(u => u.Notes);
```

3. If using `MockData` add some example objects for testing

```cs
context.UserNotes.Add(new UserNote { User = context.Users.Find(1), Note = "User specific note" });
context.AddressNotes.Add(new AddressNote { Address = context.Addresses.Find(1), Note = "Address specific note" });
```

4. Do NOT define the `UserNotes` or `AddressNotes` objects as OData EntitySets.  OData has a requirement that and entity cannot both belong to an entity set declared within the entity container and be referenced by a containment relationship.  [Reference](http://docs.oasis-open.org/odata/odata/v4.0/errata03/os/complete/part3-csdl/odata-v4.0-errata03-os-part3-csdl-complete.html#_Toc453752542)
   * If using full framework, then you WOULD define the object only as part of its parent, using the format `builder.EntitySet<User>("users").HasManyBinding<UserNote>(o => o.Notes, "notes");`, but this is not needed for ASP.Net Core.

5. Creat the API endpoint in the parent object's controller.  The parent object must have an OData EntitySet defined, and the path to the repeating object must be within the parent's path.

```cs
/// <summary>Query user notes</summary>
[HttpGet]
[ODataRoute("users({id})/notes")]
[ProducesResponseType(typeof(IEnumerable<UserNote>), 200)] // Ok
[ProducesResponseType(typeof(void), 404)]  // Not Found
[EnableQuery(AllowedQueryOptions = AllowedQueryOptions.All, MaxNodeCount = 100000)]
public async Task<IActionResult> GetNotes([FromRoute] int id) {
    var notes = _db.UserNotes;
    if (!await notes.AnyAsync(n => n.User.Id == id)) {
        return NotFound();
    }
    return Ok(notes);
}
```