# API Enum Support for ASP.Net Core

Part of a good API design is endpoint discovery and self-documentation.  Part of that includes ensuring a developer consuming the API understands the meaning of all data being retrieved or sent to the API.  This can be referred to as `Human Readable` data.  Object properties like `Status` or `Type` are frequently stored in the database as integers or even foreign keys to other tables, but exposing those integers in the API does not identify what the data represents.  For example, a developer would not know that a house `Type` of 0 means `Residential` and a house Type of 1 means `Business`.  This can be solved however through the use of enumerations.

Used properly, enumerations can allow the API to work with and enforce the allowed types in the form of “Human Readable” text while transforming back to the database’s native data type as needed.  This document will walk through an example of how to set that up in ASP.Net Core, Entity Framework, and Open API/Swagger.

1. Add the `enum` definition within the existing object class as in the following example:

```cs
    // For brevity of this document, this is a partial representation of the larger Address object
    public class Address {
        public enum AddressType {
            Residential,
            Business
        }
        public int Id { get; set; }
        public string Street { get; set; }
        public string City { get; set; }
        public string State { get; set; }
        public string ZipCode { get; set; }
        public AddressType Type { get; set; }
    }
```

2. Add to the `ApiDbContext` class, `OnModelCreating` function so Entity Framework knows to convert the data type input by the API (string) to the input type expected by the database (int).  The below example code would be added just below any existing Entity rules for the same object:

```cs
modelBuilder.Entity<Address>().Property(c => c.Type).HasConversion<int>(); // Store enum in database as an int
```

3. If using `MockData`, add the data taking advantage of the new enum definition like following example:

```cs
address.Type = Address.AddressType.Residential;
```

4. In the `Startup` class, `ConfigureServices()` function, `AddSwaggerGen()` section, add the following line to show the enum as a string in all Swagger documentation:

```cs
options.DescribeAllEnumsAsStrings();
```

At this point, the actual database value will be completly abstracted away from the API and its Swagger documentation, and all API actions will send and receive only the string representation of allowed values defined by the enum.
