# Angular Azure SSO Authentication and Authorization (angular-oauth2-oidc)

The angular-oauth2-oidc library is a client-side library for obtaining tokens from OpenID Connect providers, including Azure AD. It includes support for the OAuth2 Authorization Code Flow.

## Configuring the App Registration for Auth Code Flow

Authorization Code Flow was created with single-page apps in mind, and solves some cookie-related issues that may be encountered on certain devices with implicit flow.

1. Make sure "implict grant" is off. Due to a CORS issue on the authorization endpoint, the app registration needs to use the auth code flow exclusively.
2. Under "Expose an API", the api://{client_id}/user_impersonation scope needs to be enabled. This is the scope the UI will need to request to gain access to the API.
3. In the application manifest, the "accessTokenAcceptedVersion" property should be set to 2. Otherwise, token validation will fail in the UI and API due to the "iss" field not matching.
4. When setting up redirectUri values for the application, make sure they are associated with the "Single-page application" platform, and not the "Web Application" platform. (Authentication section of the App Registration Config)
## Configuring the angular-oauth2-oidc library

1. Add angular-oauth2-oidc to your project.
2. Add the configuration to your environment.ts (see project page for full list of options):

```javascript
authConfig: {
		issuer: "https://login.microsoftonline.com/1a9277a3-ef66-41f6-96b5-c5390ee468a7/v2.0",
		clientId: "f9c9611d-6a23-4d2e-8dce-14da56bd8acc",
		responseType: 'code',
		clearHashAfterLogin: true,
		requestAccessToken: true,
		scope: 'api://f9c9611d-6a23-4d2e-8dce-14da56bd8acc/user_impersonation profile',
		showDebugInformation: true,
		skipIssuerCheck: false,
		strictDiscoveryDocumentValidation: false
	}
```

* For the scope setting, the api scope is required for authorization in the API, and the profile scope is only required if the UI needs to display who is logged in.

3. Configure your app module:

```javascript
import { OAuthModule, OAuthModuleConfig, AuthConfig } from 'angular-oauth2-oidc';

export function oAuthModuleConfigFactory(apiUrl: string) {
    return {
        resourceServer:
        {
            allowedUrls: [apiUrl], //URL of your API
            sendAccessToken: true
        }
    };
}
```

```javascript
imports: [
    OAuthModule.forRoot()
],
providers: [
    { provide: AUTH_CONFIG, useValue: environment.authConfig },
	{ provide: API_URL, useValue: apiUrl },
    {
        provide: OAuthModuleConfig,
        useFactory: oAuthModuleConfigFactory,
        deps: [API_URL]
    },
    IdentityService
]
```

4. Initialize library
```javascript

	private loggedInSubject$: ReplaySubject<boolean> = new ReplaySubject<boolean>(1);

	private configure(authConfig: AuthConfig) {
		authConfig.redirectUri = this.origin;
		this.osvc.configure(authConfig);
		this.osvc.setupAutomaticSilentRefresh();
		this.osvc.loadDiscoveryDocumentAndTryLogin().then(res => {
			this.loggedInSubject$.next(this.osvc.hasValidAccessToken() && this.osvc.hasValidIdToken());
		});
	}
```

5. Login user
```javascript
	public login(): void {
		this.osvc.initLoginFlow();
	}

	public logout(): void {
		this.osvc.logOut();
	}
```
* If users should be automatically logged in on initialization, loadDiscoveryDocumentAndLogin() can be called instead of loadDiscoveryDocumentAndTryLogin()

6. Implement UI authorization
* If there are routes that require authentication, you can expose the logged in status, and access it in a route guard:
```javascript
public get isLoggedIn(): Observable<boolean> {
    return this.loggedInSubject$.pipe(take(1));
}
```
* If you need to access claims in the ID token:
```javascript
	public getRoles(): Observable<string[]> {
		return this.loggedInSubject$.pipe(
			take(1),
			map(loggedIn => {
				if (loggedIn) {
					let claims: any = this.osvc.getIdentityClaims() ?? {};
					return claims.roles || [];
				} else {
					return null;
				}
			})
		);
	}
```