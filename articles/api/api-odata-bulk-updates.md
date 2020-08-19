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
/// <param name="user">A full user object</param>
/// <returns>A new user</returns>
/// <response code="201">The user was successfully created</response>
/// <response code="400">The user is invalid</response>
/// <response code="401">Authentication required</response>
/// <response code="403">Access denied due to inadaquate claim roles</response>
[HttpPost]
[ODataRoute("")]
[Produces("application/json")]
[ProducesResponseType(typeof(User), 201)] // Created
[ProducesResponseType(typeof(ModelStateDictionary), 400)] // Bad Request
[ProducesResponseType(typeof(void), 401)] // Unauthorized
//[Authorize]
public async Task<IActionResult> Post([FromBody] User user) {
    try {
        if (!ModelState.IsValid) {
            return BadRequest(ModelState);
        }
        _db.Users.Add(user);
        await _db.SaveChangesAsync();
        return Created("", user);
    } catch (Exception ex) {
        return StatusCode(500, ex.Message+Constants.messageAppInsights);
    }
}
```

to this:

```cs
/// <summary>Create a new user</summary>
/// <remarks>
/// Supports either a single object or an array
/// </remarks>
/// <param name="user">A full user object</param>
/// <returns>A new user or list of users</returns>
/// <response code="201">The user was successfully created</response>
/// <response code="400">The user is invalid</response>
/// <response code="401">Authentication required</response>
/// <response code="403">Access denied due to inadaquate claim roles</response>
[HttpPost]
[ODataRoute("")]
[Produces("application/json")]
[ProducesResponseType(typeof(IEnumerable<User>), 201)] // Created
[ProducesResponseType(typeof(ModelStateDictionary), 400)] // Bad Request
[ProducesResponseType(typeof(void), 401)] // Unauthorized
//[Authorize]
public async Task<IActionResult> Post([FromBody] User user) {
    try {
        Request.Body.Position = 0;
        var streamReader = (await new StreamReader(Request.Body).ReadToEndAsync()).Trim();
        // Determine if a single object or an array was passed in
        if (streamReader.StartsWith("{")) {
            var node = JsonConvert.DeserializeObject<User>(streamReader);
            ModelState.Clear();
            TryValidateModel(node);
            if (!ModelState.IsValid) {
                return BadRequest(ModelState);
            }
            _db.Users.Add(node);
            await _db.SaveChangesAsync();
            return Created("", node);
        } else if (streamReader.StartsWith("[")) {
            var nodes = JsonConvert.DeserializeObject<IEnumerable<User>>(streamReader);
            foreach (var node in nodes) {
                ModelState.Clear();
                TryValidateModel(node);
                if (!ModelState.IsValid) {
                    return BadRequest(ModelState);
                }
                _db.Users.Add(node);
            }
            await _db.SaveChangesAsync();
            return Created("", nodes);
        } else {
            return BadRequest();
        }
    } catch (Exception ex) {
        return StatusCode(500, ex.Message+Constants.messageAppInsights);
    }
}
```

Notice how there is really no extra work to do withing the controller action other than wrapping any existing funtionality for adding a single user into a loop to execute for each user, then executing `SaveChangesAsync()` just one time after the completion of the loop.  At any time, should an error occur, none of the changes would be committed.

To support bulk PATCH, rather than modifying the exisitng single PATCH, a second action is added for bulk PATCH support as follows:

```cs
/// <summary>Bulk edit users</summary>
/// <param name="userDeltas">An array of partial user objects.  Only properties supplied will be updated.</param>
/// <returns>An updated list of users</returns>
/// <response code="200">The user was successfully updated</response>
/// <response code="400">The user is invalid</response>
/// <response code="401">Authentication required</response>
/// <response code="403">Access denied due to inadaquate claim roles</response>
/// <response code="404">The user was not found</response>
[HttpPatch]
[ODataRoute("")]
[Produces("application/json")]
[ProducesResponseType(typeof(IEnumerable<User>), 200)] // Ok
[ProducesResponseType(typeof(ModelStateDictionary), 400)] // Bad Request
[ProducesResponseType(typeof(void), 401)] // Unauthorized - User not authenticated
[ProducesResponseType(typeof(ForbiddenException), 403)] // Forbidden - User does not have required claim roles
[ProducesResponseType(typeof(void), 404)] // Not Found
//[Authorize]
public async Task<IActionResult> PatchBulk([FromBody] IEnumerable<User> userDeltas) {
    try {
        Request.Body.Position = 0;
        var patchUsers = JsonConvert.DeserializeObject<IEnumerable<dynamic>>(await new StreamReader(Request.Body).ReadToEndAsync());
        var userProperties = typeof(User).GetProperties();
        // Get list of all passed in Ids
        var idList = new List<int>();
        foreach (var patchUser in patchUsers) {
            idList.Add((int)patchUser["id"]);
        }
        _db.ChangeTracker.AutoDetectChangesEnabled = false;
        // Make one SQL call to get all the database objects to Patch
        var users = await _db.Users.Where(st => idList.Contains(st.Id)).ToListAsync();
        // Update each database objects
        foreach (var patchUser in patchUsers) {
            var user = users.Find(st => st.Id == (int)patchUser["id"]);
            if (user== null) {
                return NotFound();
            }
            foreach (var patchUserProperty in patchUser.Properties()) {
                var patchUserPropertyName = patchUserProperty.Name;
                if (patchUserPropertyName != "id") { // Cannot change the id but it will always be passed in
                    // Loop through the changed properties updating the object
                    for (int i = 0; i < userProperties.Length; i++) {
                        if (String.Equals(patchUserPropertyName, userProperties[i].Name, StringComparison.OrdinalIgnoreCase) == 0) {
                            _db.Entry(user).Property(userProperties[i].Name).CurrentValue = Convert.ChangeType(patchUserProperty.Value, userProperties[i].PropertyType);
                            _db.Entry(user).State = EntityState.Modified;
                            break;
                            // Could optionally even support deltas within deltas here
                        }
                    }
                }
            }
            _db.Users.Update(user);
        }
        await _db.SaveChangesAsync();
        _db.ChangeTracker.AutoDetectChangesEnabled = true;
        return Ok(users);
    } catch (Exception ex) {
        _telemetryTracker.TrackException(ex);
        return StatusCode(500, ex.Message + Constants.messageAppInsights);
    }
}
```

An example of how to call a bulk PATCH operation is as follows:

```json
[
    {
      "id": 1,
      "lastName": "Gilchrist2"
    },
    {
      "id": 2,
      "firstName": "Paul2"
    }
]
```
