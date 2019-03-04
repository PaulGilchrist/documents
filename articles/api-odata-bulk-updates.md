# OData Bulk Updates

Allowing applications to POST, PUT, or PATCH multiple objects in a single API request can greatly increase the performance of any API.  This document will discuss the requirements for getting this working within an OData and Open API project.

## OData Requirements

OData does not allow an array to be passed in as part of the request body, requiring a single object.  If attempting to pass in an array, it will fail validation and the controller action's input parameter will be null regardless of the actual request body content.  OData performs similarly for GET requests even when a results array is expected.  For example, if making a get request for all user objects, OData will actually return a single object which contains the actual array in a value property as in the below example JSON:

```json
{
  "@odata.context": "https://localhost:44375/odata/$metadata#users",
  "value": [
    {
      "id": 1,
      "firstName": "Paul",
      "middleName": null,
      "lastName": "Gilchrist",
      "email": "paul.gilchrist@outlook.com",
      "phone": "123-555-4567"
    },
    {
      "id": 2,
      "firstName": "John",
      "middleName": null,
      "lastName": "Hicks",
      "email": "John.Hicks@Skyvu.com",
      "phone": "(402) 129-8483"
    }
  ]
}
```

This is the same format we should use when doing bulk POST, PUT, or PATCH actions.  To support bulk updates for the above `User` object model, we would start by adding a `UserList` model as follows:

```cs
public class UserList {
    public List<User> value { get; set; }
}
```

Next, your single object POST (for example) would be changed from this:

```cs
/// <summary>Create a new user</summary>
/// <remarks>
/// Make sure to secure this action before production release
/// </remarks>
/// <param name="user">A full user object</param>
[HttpPost]
[ODataRoute("")]
[ProducesResponseType(typeof(User), 201)] // Created
[ProducesResponseType(typeof(ModelStateDictionary), 400)] // Bad Request
[ProducesResponseType(typeof(void), 401)] // Unauthorized
//[Authorize]
public async Task<IActionResult> Post([FromBody] User user) {
    if (!ModelState.IsValid) {
        return BadRequest(ModelState);
    }
    _db.Users.Add(user);
    await _db.SaveChangesAsync();
    return Created("", user);
}
```

to this:

```cs
/// <summary>Create one or more new users</summary>
/// <remarks>
/// Make sure to secure this action before production release
/// </remarks>
/// <param name="userList">An object containing an array of full user objects</param>
[HttpPost]
[ODataRoute("")]
[ProducesResponseType(typeof(IEnumerable<User>), 201)] // Created
[ProducesResponseType(typeof(ModelStateDictionary), 400)] // Bad Request
[ProducesResponseType(typeof(void), 401)] // Unauthorized
//[Authorize]
public async Task<IActionResult> Post([FromBody] UserList userList) {
    // Swagger will give error if not using options.CustomSchemaIds((x) => x.Name + "_" + Guid.NewGuid());
    if (!ModelState.IsValid) {
        return BadRequest(ModelState);
    }
    var users = userList.value;
    foreach (User user in users) {
        _db.Users.Add(user);
    }
    await _db.SaveChangesAsync();
    return Created("", users);
}
```

Notice how there is really no extra work to do withing the controller action other than wrapping any existing funtionality for adding a single user into a loop to execute for each user, then executing `SaveChangesAsync()` just one time after the completion of the loop.  At any time, should an error occur, none of the changes would be committed.

This approach work for both POST and PATCH actions but presents an issue for Swashbuckle.  Swashbuckle thinks since theis is a POST action for user, that a single user will be returned, but what is actually being returned is the array of created users the same as was returned for a GET action.  This causes Swashbuckle to try and define a second schema for the user object, causing a schema ID conflict.  The solution for this is to edit the `startup.cs` file, `ConfigureServices()` function, `AddSwaggerGen()` section adding the following line to ensure unique schema's are generated in these scenarios:

```cs
options.CustomSchemaIds((x) => x.Name + "_" + Guid.NewGuid());
```

We now have both OData and Swashbuckle/Swagger working properly for both POST and PUT actions, but there is still an issue when doiung a PATCH action due to OData's `Delta<T>` requirements.  A `Delta<T>` cannot be defined within another object, and also does not support a partial object below the root level object.  Both of these issues can be solved by using a dynamic object instead.  The first step is creating a new class in the `Models` folder as follows:

```cs
using System.Collections.Generic;

namespace API.Models {
    public class DynamicList {
        public List<dynamic> value { get; set; }
    }
}
```

Being that this class is dynamic, it can be used by all controller PATCH actions.

A previous single object PATCH action would then change from this example:

```cs
/// <summary>Edit the user with the given id</summary>
/// <remarks>
/// Make sure to secure this action before production release
/// </remarks>
/// <param name="id">The user id</param>
/// <param name="userDelta">A partial user object.  Only properties supplied will be updated.</param>
[HttpPatch]
[ODataRoute("({id})")]
[ProducesResponseType(typeof(User), 200)] // Ok
[ProducesResponseType(typeof(ModelStateDictionary), 400)] // Bad Request
[ProducesResponseType(typeof(void), 401)] // Unauthorized
[ProducesResponseType(typeof(void), 404)] // Not Found
//[Authorize]
public async Task<IActionResult> Patch([FromRoute] int id, [FromBody] Delta<User> userDelta) {
    if (!ModelState.IsValid) {
        return BadRequest(ModelState);
    }
    var dbUser = _db.Users.Find(id);
    if (dbUser == null) {
        return NotFound();
    }
    _db.Entry(dbUser).State = EntityState.Detached;
    // Loop through the changed properties updating the user object
    userDelta.Patch(dbUser);
    await _db.SaveChangesAsync();
    return Ok(dbUser);
}
```

to this bulk PATCH implementation:

```cs
/// <summary>Bulk edit users</summary>
/// <remarks>
/// Make sure to secure this action before production release
/// </remarks>
/// <param name="userList">An object containing an array of partial user objects.  Only properties supplied will be updated.</param>
[HttpPatch]
[ODataRoute("")]
[ProducesResponseType(typeof(IEnumerable<User>), 200)] // Ok
[ProducesResponseType(typeof(ModelStateDictionary), 400)] // Bad Request
[ProducesResponseType(typeof(void), 401)] // Unauthorized
[ProducesResponseType(typeof(void), 404)] // Not Found
//[Authorize]
public async Task<IActionResult> Patch([FromBody] UserList userList) {
    // Swagger will report a UserList object model, but what is actually being passed in is a dynamic list since PATCH does not require the full object properties
    //     This mean we actually need a DynamicList, so reposition and re-read the body
    //     Full explaination ... https://github.com/PaulGilchrist/documents/blob/master/articles/api-odata-bulk-updates.md
    Request.Body.Position = 0;
    var patchUserList = JsonConvert.DeserializeObject<DynamicList>(new StreamReader(Request.Body).ReadToEnd());
    var patchUsers = patchUserList.value;
    List<User> dbUsers = new List<User>(0);
    System.Reflection.PropertyInfo[] userProperties = typeof(User).GetProperties();
    foreach (JObject patchUser in patchUsers) {
        var dbUser = _db.Users.Find((int)patchUser["id"]);
        if (dbUser == null) {
            return NotFound();
        }
        var patchUseProperties = patchUser.Properties();
        // Loop through the changed properties updating the user object
        foreach (var patchUserProperty in patchUseProperties) {
            foreach (var userProperty in userProperties) {
                if (String.Compare(patchUserProperty.Name, userProperty.Name, true) == 0) {
                    _db.Entry(dbUser).Property(userProperty.Name).CurrentValue = Convert.ChangeType(patchUserProperty.Value, userProperty.PropertyType);
                    // Could optionally even support delta's within delta's here
                }
            }
        }
        _db.Entry(dbUser).State = EntityState.Detached;
        _db.Users.Update(dbUser);
        dbUsers.Add(dbUser);
    }
    await _db.SaveChangesAsync();
    return Ok(dbUsers);
}
```

With the above code, Open API/Swagger will document a `UserList` object model, but what is actually being passed in is a `DynamicList` since `PATCH` only passes in the properties that have changed.  If we were to use the `UserList` instead of `DynamicList`, properties not passed in would show as null or zero, and it would be impossible to tell what parameters require updating.  For this reason, we need to re-position and re-read the request body into a `DynamicList` object, pull out the user array, looping through it, updating only those properties passed in.  This extends OData’s `Delta<T>` object not only in the support of bulk updates, but also in the optional ability to patch nested complex objects.

An example of how to call a bulk PATCH operation is as follows:

```json
{
  "value": [
    {
      "id": 1,
      "lastName": "Gilchrist2"
    },
    {
      "id": 2,
      "firstName": "John2"
    }
  ]
}
```

As with POST and PUT, the array of objects is contained within the root object's `value` property, with only the `id` required.  All the remaining properties are optional and should only be passed in if they have changed.