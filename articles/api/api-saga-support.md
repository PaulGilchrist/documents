# API SAGA Support

It is important that both data persistence and an event message notification of that persistence either both succeed or both fail.  This can be accomplished through the SAGA compensation pattern if a step fails you compensate by rolling back the previous states.  Many times, rolling back is not possible, but instead that previous change can be compensated for by an equal but opposite change.  For example, an object creation would be compensated for with deleting that object.  An object update would be compensated for with am update back to the original values.  And finally, an object delete would require re-creating the object preferably with the original primary key or identifier.

The following code examples show how this can be accomplished.  These examples are from the GitHub project [saga-api](https://github.com/PaulGilchrist/saga-api), with some code and comments removed to help focus in on the SAGA compensation pattern specifically.

## HTTP POST (create/insert new object)

```cs
public async Task<IActionResult> Post([FromBody] Contact contact) {
    Contact newContact;
    try {
        newContact = await _contactService.Create(contact);
    } catch(Exception ex) {
        return StatusCode(500,ex.Message);
    }
    try {
        _messageService.Send(_queueName, "created", newContact);
        return Created("",newContact);
    } catch(Exception ex) {
        // Compensation to rollback POST
        await _contactService.Remove(newContact.Id);
        return StatusCode(500,ex.Message);
    }
}
```

## HTTP PATCH/PUT (update existing object)

```cs
public async Task<IActionResult> Patch([FromRoute] string id,[FromBody] dynamic contactDelta) {
    Contact? foundContact = null;
    Contact? updatedContact = null;
    try {
        foundContact = await _contactService.Get(id);
        if(foundContact == null) {
            return NotFound();
        }
        contactDelta = JsonConvert.DeserializeObject<dynamic>(Convert.ToString(contactDelta));
        updatedContact = API.Classes.Delta.Patch<Contact>(contactDelta,foundContact);
        await _contactService.Update(id,updatedContact);
    } catch(Exception ex) {
        return StatusCode(500,ex.Message);
    }
    try {
        _messageService.Send(_queueName, "updated", updatedContact);
        return NoContent();
    } catch(Exception ex) {
        // Compensation to rollback PATCH
        await _contactService.Update(id,foundContact);
        return StatusCode(500,ex.Message);
    }
}
```

## HTTP DELETE (delete existing object)

```cs
public async Task<IActionResult> Delete([FromRoute] string id) {
    Contact foundContact;
    try {
        foundContact = await _contactService.Get(id);
        if(foundContact == null) {
            return NotFound();
        }
        await _contactService.Remove(id);
    } catch(Exception ex) {
        return StatusCode(500,ex.Message);
    }
    try {
        _messageService.Send(_queueName, "deleted", id);
        return NoContent();
    } catch(Exception ex) {
        // Preferable this will allow using the original primary key / identifier
        var newContact = await _contactService.Create(foundContact);                
        return StatusCode(500,ex.Message);
    }
}
```

If using OData or similar and supporting batch transactions, see the document named [OData $batch SAGA Support](https://github.com/PaulGilchrist/documents/blob/master/articles/api/api-odata-%24batch-saga-support.md)