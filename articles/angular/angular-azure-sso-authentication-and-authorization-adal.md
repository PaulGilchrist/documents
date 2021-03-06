# Angular Azure SSO Authentication and Authorization (adal-angular4)

## Steps to implement Azure SSO Authentication and Authorization in Angular

1. Request Azure team create an app registration for each environment (dev, qa, staging, production)

    * Azure team will supply you with a clientId for each app registration created
    * If you know what application roles will be needed, supply their case sensitive names with the request
        * This can also be done at any later time
    * If you know what Active Directory users or groups need to belong to each role, supply this with the request
        * This can also be done at any later time
2. Add `adal-angular4` to `project.json`, and `npm install` it into the project
3. Add the following to the `app.module.ts`.  When adding this code, do not remove any existing providers.

```javascript
import { AdalService, AdalGuard } from 'adal-angular4';
providers: [
	AdalService,
	AdalGuard,
]
```

4. Can optionally add `canActivate: [AdalGuard]` to any route objects
5. Add the Azure app registration's clientIds and tennant.  Change the below example to use the clientIds provided for each environment.  The remaining values do not change.

```javascript
azureAuthProvider: {
	clientId: 'bd065891-b008-4968-9b26-5f2bcb9c1b66',
	tenant: 'pulte.onmicrosoft.com'
}
```

6. Add the following to the `app.component.ts`.  When adding this code, do not remove any existing parameters or functionality in the constructor

```javascript
import { AdalService } from 'adal-angular4';
import { environment } from './../environments/environment';

constructor(private adalService: AdalService) {
	// init requires object with clientId and tenant properties
	adalService.init(environment.azureAuthProvider);
}
```

7. Add the following to the main navigation bar component (usually `nav-top.component.ts`).  Do not remove any additional parameters or functionality in the constructor or `ngOnInit` functions

```javascript
import { AdalService } from 'adal-angular4';
constructor(private adalService: AdalService) {}
ngOnInit(): void {
	this.adalService.handleWindowCallback();
}
login(): void {
	this.adalService.login();
}
logout(): void {
	this.adalService.logOut();
}
get authenticated(): boolean {
	return this.adalService.userInfo.authenticated;
}
```

8. Add the following to the main navigation bar component's HTML template (usually `nav-top.component.html`).  When adding this code, do not remove any existing parameters or functionality in the constructor or `ngOnInit` functions

```javascript
<form class="navbar-form navbar-right">
	<input *ngIf="!authenticated" (click)="login()" type="button" class="btn btn-primary" value="Login">
	<input *ngIf="authenticated" (click)="logout()" type="button" class="btn sm btn-default" value="Logout">
</form>
<div *ngIf="authenticated" class="nav navbar-right hidden-xs hidden-sm">
	<span class="navbar-text">{{ adalService.userInfo.profile.given_name + ' ' + adalService.userInfo.profile.family_name }}</span>
</div>
```

9. You can also optionally show/hide navigation items using standard `*ngIf="authenticated"`
10. Optionally, if wanting helper functions for extending ADAL capabilities, add an `identity.service.ts` file to the `app.module` containing the following additional functions:

```javascript
import { Injectable } from '@angular/core';
import { AdalService } from 'adal-angular4';
@Injectable()
export class IdentityService {
	// Adds some minor functionality to the AdalService
	constructor(private adalService: AdalService) {}
	public getRoles(): string {
		if (this.adalService.userInfo.authenticated) {
		return this.adalService.userInfo.profile.roles;
		} else {
		return '';
		}
	}
	public isInAllRoles(...neededRoles: Array<string>): boolean {
		if (this.adalService.userInfo.authenticated) {
		const roles = this.adalService.userInfo.profile.roles.toLowerCase();
		return neededRoles.every(neededRole => roles.includes(neededRole.toLowerCase()));
		} else {
		return false;
		}
	}
	public isInRole(neededRole: string): boolean {
		if (this.adalService.userInfo.authenticated) {
		const roles = this.adalService.userInfo.profile.roles.toLowerCase();
		return roles.includes(neededRole.toLowerCase());
		} else {
		return false;
		}
	}
}
```

11. AdalGuard's default implementation only ensure the user is authenticated when added to an Angular route.  If wanting to extend this functionality, leverage the `isInRole()` or `isInAllRoles()` functions available in `identity.service.ts` just created.
12. You now have full authentication/authorization capabilities in Angular as well as access to the bearer token itself and all its properties through `adalService.userInfo.profile`

## Steps to add the OAuth Bearer Token to all HTTP requests in Angular

After first setting up Azure SSO Authentication and Authorization, you will be able to optionally setup Angular to intercept all HTTP requests and add the authenticated user's OAuth Bearer Token to the request using the following steps:

1. Ensure Azure SSO Authentication and Authorization has first been setup
2. Create an `auth.interceptor.ts` file in the app.module containing the following code:

```javascript
import {Injectable} from '@angular/core';
import {HttpEvent, HttpInterceptor, HttpHandler, HttpRequest, HttpResponse} from '@angular/common/http';
import { Observable } from 'rxjs';
import { AdalService } from 'adal-angular4';
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
	constructor(private adalService: AdalService) {}
	intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
		if (this.adalService.userInfo.authenticated) {
		// Clone the request to add the new header.
		const authReq = req.clone({setHeaders: { Authorization: 'Bearer ' + this.adalService.userInfo.token }});
		// Pass on the cloned request instead of the original request.
		return next.handle(authReq);
		} else {
		return next.handle(req);
		}
	}
}
```

3. Add the following to `the app.module.ts`.  When adding this code, do not remove any existing providers.

```javascript
import { HttpClientModule, HTTP_INTERCEPTORS } from '@angular/common/http';

providers: [
    { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true, }
]
```

4. Optionally create a response intercept following the same method that can watch for a `401 Unauthorized` and login the user and retry the HTTP request
