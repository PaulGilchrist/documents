# API Data Export Efficiency

Although OData is great for reusable API and complex queries, it is inefficient and incapable of returning millions of rows of data in a single request as may be needed by a data export API.  The best method of facilitating this while maintaining efficient memory usage is to stream/chunk the HTTP response rather than sending back the full response in a single packet.

The method for achieving this is very simple and testing has shown equal performance between Entity Framework and ADO.Net and even between JSON and less verbose formats like CSV.  For these reasons, the below code sticks to those simpler standards maintaining the ability to extend into deeper nested exports in the future.

## Example Code

```cs
/// <summary>Export contacts</summary>
/// <returns>A full list of contacts</returns>
/// <response code="200">The contacts were successfully retrieved</response>
[HttpGet("contacts/export")]
[Produces("application/json")]
[ProducesResponseType(typeof(IEnumerable<Contact>),200)] // Ok
public IEnumerable<Contact> Export() {            
    foreach(var contact in _db.Contacts.AsNoTracking()) {
        yield return contact;
    }
}
```

The above example code was tested with both the API and database in Azure PaaS, but the resulting JSON streaming back to a test laptop over a 200mb Internet connection.  This testing resulted in an average performance of 85 seconds per million records with tests as large as 6 million records, although total record count should not be a limiting factor.

As shown in the above code, when using Microsoft Entity Framework, and large data sets that will not be modified, it is important to use `.AsNoTracking()`.  This code loops through all contacts using `yield` to return each one separately.  This allows the HTTP pipeline to combine contact objects back together into `chunks` returning those chunks one by one to the requestor.  These chunks can then be processed as they come into the client as in the following JavaScript example code:

```js
let url = "https://api.company.com/contacts/export";
https.get(url,(res) => {
    let body = "";
    res.on("data", (chunk) => {
        body += chunk;
    });
    res.on("end", () => {
        try {
            let json = JSON.parse(body);
            // do something with JSON
        } catch (error) {
            console.error(error.message);
        };
    });
}).on("error", (error) => {
    console.error(error.message);
});
```