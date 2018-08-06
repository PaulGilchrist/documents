# Azure App Registration Setup
(Including manifest roles, and group to role assignments)

1. Go to https://portal.azure.com and choose to add a new `App Registration` giving it a name that identifies both the application and environment (ex: myApp-Dev)
2. Add a `Reply URL` pointing to the root of the website.  For the development environment, there would be both a reply URL for the Azure hosted website, and another for local development (usually http://localhost:4200 )
3. Record the generated `Application ID` which will later be used in the application as the `clientId`
4. If the application will include user roles, then open the `manifest`, and add to the `appRoles` array using the below format, but replacing all the values.  The `value` property appears in the role claim.  The `id` property is the unique identifier for the defined role, and must be a newly generated GUID.

```json
{
   "allowedMemberTypes": [
      "User"
   ],
   "displayName": "Admin",
   "id": "8ba3847b-acac-47bc-863e-b9c8556b8be1",
   "isEnabled": true,
   "description": "Admins can manage roles and perform all task actions.",
   "value": "Admin"
}
```

5. Select a user or group that will be assigned that role, choose `Directory Role`, and then choose `Add Role`.  These groups are synchronized from Active Directory, and if they do not already exist, you may need to request their creation, and population with the required user accounts.

## Action Items
* Ensure these steps still work even when the manifest has `oauth2AllowImplicitFlow` configured to false. 
