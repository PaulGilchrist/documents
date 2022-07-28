# Authentication - Supporting Multiple Identity Providers

If there is a need to support other companies accessing your API, this can be accomplished by extending the API to support more than one identity provider.  This will allow those external companies to authenticate their users, while our API trusts that authentication and only handles authorization.  This is a safer approach than issuing a non-expiring API key of client secret to that external company that might get compromised.  Authorization can be at any level, for example:  application, user, or claim role, just as it would be for any token issued by the default internal identity provider.

## Terminology

* Identity Provider = Authority = iss
* Application = Audience = aud

## API Setup and inner authorization workings

* External company's identity provider and application are added as trusted for authentication
* If the API receives a token for any trusted iss/aud combination and successfully validates that token using its respective identity provider, the user associated with that token is authenticated and a user identity is created in memory
* The application's permissions to the API are then looked up from cache and added as role claims to the user identity
* Each API endpoint authorizes or denies access based on if the user identity has the needed permission
* Expanding from one object to another is also restricted or allowed based on these same permissions

## External Company Workflow

* An external user logs into their application and is authenticated and authorized accordingly
  * At this point, the external company is already using an OAuth token representing the user, or one is generated
* The external user accesses a feature in the external application that requires data from our API
* The external application makes a call to our internal API passing the external user's OAuth token
* The internal API validates this token was signed by the external identity provider thereby ensuring the user has been authenticated successfully by the external company.  We never have access to the user's password, or any other user details not explicitly added to the token.
* The token's signature also matches the content (hash) of the token, ensuring it has not been altered since signed
* The internal API knows the user is who they say they are, and has a valid token for use of the external application, and thereby also the internal API endpoints granted access to the external application or to the external user directly.

## API Code Changes

### launchSettings.json
  * Includes comma separated list of OAuth audiences and their matching authority URLs

### ApplicationSettings.cs
  * Retrieves OAuth audiences and their matching authority URLs from the environment at launch splitting them into arrays with aligning indexes

### Program.cs
  * Sets the default authentication and challenge schemes
  * Loops through the list of audiences enabling them for JWT-bearer authentication
    * Validates the token using the authority URL
* Adds all OAuth audiences to the AuthorizationPolicyBuilder
* Add JWT token support to the Swagger UI
* Enables authentication and authorization