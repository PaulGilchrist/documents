# Angular Template Setup - Step by Step

The following steps can be used to start a new Angular project and ensure you are following current Angular best practices.  Do not use existing Visual Studio Angular project templates as they are almost always several versions out of date, and bring in .Net Core standards and overhead not native to Angular.  If desired, Visual Studio can still be used to develop this application after the template initial setup, however VS Code is a more common Angular development environment.

## Steps

1. Ensure all tools are up to date (see [Template Setup Step-by-Step](https://github.com/PaulGilchrist/documents/blob/master/articles/angular/angular-template-setup-step-by-step.md))
   * node, npm, vs code, git, @angular/cli, typescript, etc.

```cmd
ng new purchase-pro
```

2. Setup `.editorconfig` standards
3. Add `/src/server/node_modules` to `.gitignore`
4. Add PWA support

```cmd
ng add @angular/pwa --project *project-name*
```

5. Add `src/sw-worker.js` to `angular.json` assets, and copy file to `src` folder
   * This allows notifications to work even when application is not already running

6. Add `src/web.config` to `angular.json` assets, and create file to `src` folder (content below).  This will allow IIS to send the SPA app to the requestor should they GET any sub-page either existing on not found.  This allows routing to remain in the client

```xml
<configuration>
    <system.webServer>
        <rewrite>
            <rules>
                <clear />
                <rule name="Angular Routes" stopProcessing="true">
                    <match url=".*" />
                    <conditions logicalGrouping="MatchAll">
                        <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
                        <add input="{REQUEST_FILENAME}" matchType="IsDirectory" negate="true" />
                    </conditions>
                    <action type="Rewrite" url="/" />
                </rule>
            </rules>
        </rewrite>
        <staticContent>
            <mimeMap fileExtension=".json" mimeType="application/json" />
            <mimeMap fileExtension=".webmanifest" mimeType="application/json" />
            <mimeMap fileExtension=".woff" mimeType="font/woff" />
            <mimeMap fileExtension=".woff2" mimeType="font/woff2" />
        </staticContent>
    </system.webServer>
</configuration>
```

7. Add the following packages

```cmd
npm install angular-connectivity adal-angular4 animate.css applicationinsights-js bootstrap font-awesome jquery ngx-toastr

npm install @types/applicationinsights-js @types/bootstrap @types/jquery --save-dev
```

8. Add the following to `build.scripts` and `test.scripts` in the file `angular.json`

```json
"node_modules/jquery/dist/jquery.js",
"node_modules/bootstrap/dist/js/bootstrap.js",
```

9. Add the following to `build.configurations` in the file `angular.json`

```json
"qa": {
    "fileReplacements": [
        {
            "replace": "src/environments/environment.ts",
            "with": "src/environments/environment.qa.ts"
        }
    ]
},
"staging": {
    "fileReplacements": [
        {
            "replace": "src/environments/environment.ts",
            "with": "src/environments/environment.staging.ts"
        }
    ],
    "optimization": true,
    "outputHashing": "all",
    "sourceMap": false,
    "extractCss": true,
    "namedChunks": false,
    "aot": true,
    "extractLicenses": true,
    "vendorChunk": false,
    "buildOptimizer": true
},
```

10. Add the following to `serve.configurations` in the file `angular.json`.  Replace `<appname>` appropriatly.

```json
"qa": {
    "browserTarget": "<appname>:build:qa"
},
"staging": {
    "browserTarget": "<appname>:build:staging"
},
```

11. Add the following `imports` in the file `src/app/app.module.ts`

```js
    BrowserAnimationsModule,
    HttpClientModule,
    ToastrModule.forRoot(),
    ServiceWorkerModule.register('sw-worker.js', { enabled: environment.production }) // Replaced ngsw-worker.js to add ability to open application from notification
```

12. Add the following `providers` in the file `src/app/app.module.ts`

```js
    AdalService,
    AdalGuard,
    AppInsightsService,
    ConnectivityService
```

13. Raise max-line-length to 350 in file `tslint.json`

14. Setup app setting in the folder `environments` the files `environment.ts`, `environment.qa.ts`, `environment.staging.ts`, and `environment.prod.ts` following the below template, but changing the values as needed.

```js
// The file contents for the current environment will overwrite these during build.
// The build system defaults to the dev environment which uses `environment.ts`, but if you do
// `ng build --env=prod` then `environment.prod.ts` will be used instead.
// The list of which env maps to which file can be found in `.angular-cli.json`.
export const environment = {
    apiUrl: './assets/', // Would normally be a remote API, but for this demo we are using local JSON files
    appInsights: {
        instrumentationKey: '2ca976b1-98fd-4564-a913-967c14b3a19b'
    },
    azureAuthProvider: {
        aadInstance: 'https://login.microsoftonline.com/{0}',
        clientId: 'bd065891-b008-4968-9b26-5f2bcb9c1b66',
        domainHint: 'pulte.com',
        tenant: 'pulte.onmicrosoft.com'
    },
    dataCaching: { // milliseconds data is allowed to remain cached before next request for that data re-retrieves it from the remote data source
        defaultLong: 86400000, // Default for data that rarely or never changes such as a list of states (one day)
        defaultShort: 60000, // Default for data that changes frequently, but still worth caching (one minute)
        userData: 600000 // 10 minutes
    },
    envName: 'dev',
    vapid: {
        publicKey: 'BE6IqM0la0Mr7jg5w5vYwPk5gwbypKcpsJrqH-xX3nfLm9BCCrt2EDCyMZH7yZYFDGtGxtCgZqCgnGeuJehndoc'
    },
    production: false
};
```

15. Create folder `src/app/components/app` and move the following files into the new folder
    * `app.component.html`, `app.component.scss`, `app.component.spec.ts`, and `app.component.ts`

16. Create folder `src/app/services` and copy the following files into the new folder (get from template project or create new)
    * `app-insights.service.ts` - This service allows client side custom telemetry logging to Azure App Insights
    * `auth.interceptor.ts` - This service allows automatically adding the current Oauth bearer token to all HTTP request
    * `identity.service.ts` - This service extends AdalAngular.js functionality allowing for easy checks of a user's roles
    * `logging.interceptor.ts` - This service automatically logs any HTTP related errors to the console

17. Add the above two interceptors to the `providers` section of the file `app.module.ts`

```js
{ provide: HTTP_INTERCEPTORS, useClass: LoggingInterceptor, multi: true }, // Time how long each http requests takes
{ provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true } // Add token to all API requests
```

18. Add the following imports to the file `src/styles.scss`

```css
@import '~animate.css/animate.css';
@import '~bootstrap/scss/bootstrap.scss';
@import '~font-awesome/css/font-awesome.css';
@import '~ngx-toastr/toastr-bs4-alert';
```

19. Optional - Add the following to the `Application Imports` section of the file `src/polyfills.ts`

20. Add the following code to the `AppComponent` to initialize `adalAngular.js`, force login, and send telemetry to `Azure App Insights`

```js
constructor(private adalService: AdalService, private appInsightsService: AppInsightsService, private router: Router) {
    // init requires object with clientId and tenant properties
    adalService.init(environment.azureAuthProvider);
}

ngOnInit(): void {
    this.adalService.handleWindowCallback();
    if (this.adalService.userInfo.authenticated) {
        const url = localStorage.getItem('url');
        if (url != null) {
            localStorage.removeItem('url');
            this.router.navigateByUrl(url);
        }
    } else {
        localStorage.setItem('url', this.router.url);
        this.adalService.login();
    }
    this.appInsightsService.logPageView('app.component', '/');
}
```

## Create First Module

The template's `App` module should not contain any application code not shared across all modules within the application.  Application development should begin with creation of the first feature module as follows:

1. Create folder `srs/app/<modulename>-module` and include the sub-folders `components`, `models`, and `services`

2. Create folder `srs/app/<modulename>-module/components/home` containing the files `home.component.html`, `home.component.scss`, and `home.component.ts`.  If the module will have no default route, then this component can be given a different name.

3. Copy all content out of `src/app/components/app.component.html` except the `<router-outlet></router-outlet>` and paste it into `src/app/<modulename>-module/components/home/home.component.html`

4. Replace `src/app/components.app.component.html` with the following code

```js
<div class="container-fluid">
    <main>
        <router-outlet></router-outlet>
    </main>
</div>
```

5. Add the following code to the file `src/app/<modulename>-module/components/home/home.component.html`.  Replace `<modulename>` and `<moduleroute>` appropriatly.

```js
import { Component, OnInit } from '@angular/core';
import { Router } from '@angular/router';

import { AppInsightsService } from '../../../services/app-insights.service';

@Component({
  // tslint:disable-next-line: component-selector
  selector: '<modulename>-home',
  styleUrls: ['./home.component.scss'],
  templateUrl: './home.component.html'
})
export class HomeComponent implements OnInit {

    constructor(public router: Router, private appInsightsService: AppInsightsService) {}

    ngOnInit() {
        // Example of how to add Application Insights tracking to a component
        this.appInsightsService.logPageView('<modulename>.home.component', '/<moduleroute>');
    }

}
```

6. Create file `srs/app/<modulename>-module/<modulename>.module.ts`.  Replace `<modulename>` appropriatly.

```js
import { NgModule } from '@angular/core';
import { RouterModule } from '@angular/router';
import { CommonModule, LocationStrategy, HashLocationStrategy } from '@angular/common';

import * as $ from 'jquery';

/* Module Declarations */
import { HomeComponent } from './components/home/home.component';

@NgModule({
    declarations: [
        HomeComponent
    ],
    imports: [
        CommonModule,
        RouterModule.forChild([
            { path: '', component: HomeComponent },
        ])
    ],
    providers: [
        { provide: LocationStrategy, useClass: HashLocationStrategy }
    ],

})
export class <modulename>Module {}
```

7. Configure file `src/app/components/app-routing.module.ts` to route to the new sub-module's home page as follows.  Replace `<modulename>` and `<moduleroute>` appropriatly.
```js
    { path: '', redirectTo: '/<modulepath>', pathMatch: 'full' },
    { path: '<moduleroute>', loadChildren: () => import('./<modulename>-module/<modulename>.module').then(m => m.<modulename>Module) },
```

## Create First Component - OAuth Token Test Page

This component is not meant to remain all the way through to production, but makes for a good first component, and testing tool to ensure OAuth Token security is setup properly.

1. Create folder `srs/app/components/token` containing the files `token.component.html`, `token.component.scss`, and `token.component.ts`.

2. Add the following code to the file `src/app/components/token.component.html`.

```js
<div *ngIf='adalService.userInfo.profile' class="card animated slideInUp">
    <div class="card-body">
        <a href="#tokenView">
            <h4>{{ adalService.userInfo.profile.given_name + ' ' + adalService.userInfo.profile.family_name }}</h4>
        </a>
        <div class="form-group row">
            <label for="aud" class="col-form-label col-sm-3 text-info label-tooltip" tooltip="Audience of the token. When the token is issued to a client application, the audience is the client_id of the client.">Audience
                ID</label>
            <input type="text" name="aud" class="form-control col-sm-9" disabled [value]="adalService.userInfo.profile.aud">
        </div>
        <div class="form-group row">
            <label for="iss" class="col-form-label col-sm-3 text-info label-tooltip" tooltip="Identifies the token issuer">Issuer</label>
            <input type="text" name="iss" class="form-control col-sm-9" disabled [value]="adalService.userInfo.profile.iss">
        </div>
        <div class="form-group row">
            <label for="iat" class="col-form-label col-sm-3 text-info label-tooltip" tooltip="Issued at time. The time when the JWT was issued. The time is represented as the number of seconds from January 1, 1970 (1970-01-01T0:0:0Z) UTC until the time the token was issued.">Issued
                At</label>
            <input type="text" name="iat" class="form-control col-sm-9" disabled [value]="getDateString(adalService.userInfo.profile.iat)">
        </div>
        <div class="form-group row" *ngIf="adalService.userInfo.profile.nbf">
            <label for="nbf" class="col-form-label col-sm-3 text-info label-tooltip" tooltip="Not before time. The time when the token becomes effective. For the token to be valid, the current date/time must be greater than or equal to the Nbf value. The time is represented as the number of seconds from January 1, 1970 (1970-01-01T0:0:0Z) UTC until the time the token was issued.">Not
                Valid Before</label>
            <input type="text" name="nbf" class="form-control col-sm-9" disabled [value]="getDateString(adalService.userInfo.profile.nbf)">
        </div>
        <div class="form-group row">
            <label for="exp" class="col-form-label col-sm-3 text-info label-tooltip" tooltip="Expiration time. The time when the token expires. For the token to be valid, the current date/time must be less than or equal to the exp value. The time is represented as the number of seconds from January 1, 1970 (1970-01-01T0:0:0Z) UTC until the time the token was issued.">Expires</label>
            <input type="text" name="exp" class="form-control col-sm-9" disabled [value]="getDateString(adalService.userInfo.profile.exp)">
        </div>
        <div class="form-group row" *ngIf="adalService.userInfo.profile.ver">
            <label for="ver" class="col-form-label col-sm-3 text-info label-tooltip" tooltip="Version. The version of the JWT token, typically 1.0.">Version</label>
            <input type="text" name="ver" class="form-control col-sm-9" disabled [value]="adalService.userInfo.profile.ver">
        </div>
        <div class="form-group row" *ngIf="adalService.userInfo.profile.tid">
            <label for="tid" class="col-form-label col-sm-3 text-info label-tooltip" tooltip="Tenant identifier (ID) of the Azure AD tenant that issued the token.">Tenant
                Identifier</label>
            <input type="text" name="tid" class="form-control col-sm-9" disabled [value]="adalService.userInfo.profile.tid">
        </div>
        <div class="form-group row" *ngIf="adalService.userInfo.profile.amr">
            <label for="amr" class="col-form-label col-sm-3 text-info">Amr</label>
            <input type="text" name="amr" class="form-control col-sm-9" disabled [value]="adalService.userInfo.profile.amr">
        </div>
        <div class="form-group row">
            <label for="roles" class="col-form-label col-sm-3 text-info">Roles</label>
            <input type="text" name="roles" class="form-control col-sm-9" disabled [value]="adalService.userInfo.profile.roles">
        </div>
        <div class="form-group row" *ngIf="adalService.userInfo.profile.oid">
            <label for="oid" class="col-form-label col-sm-3 text-info label-tooltip" tooltip="Object identifier (ID) of the user object in Azure AD.">Object
                Identifier</label>
            <input type="text" name="oid" class="form-control col-sm-9" disabled [value]="adalService.userInfo.profile.oid">
        </div>
        <div class="form-group row" *ngIf="adalService.userInfo.profile.email">
            <label for="upn" class="col-form-label col-sm-3 text-info label-tooltip" tooltip="Email Address">Email</label>
            <input type="text" name="email" class="form-control col-sm-9" disabled [value]="adalService.userInfo.profile.email">
        </div>
        <div class="form-group row" *ngIf="adalService.userInfo.profile.upn">
            <label for="upn" class="col-form-label col-sm-3 text-info label-tooltip" tooltip="User principal name of the user.">User
                Principal Name</label>
            <input type="text" name="upn" class="form-control col-sm-9" disabled [value]="adalService.userInfo.profile.upn">
        </div>
        <div class="form-group row" *ngIf="adalService.userInfo.profile.unique_name">
            <label for="unique_name" class="col-form-label col-sm-3 text-info label-tooltip" tooltip="A unique identifier for that can be displayed to the user. This is usually a user principal name (UPN).">Unique
                Name</label>
            <input type="text" name="unique_name" class="form-control col-sm-9" disabled [value]="adalService.userInfo.profile.unique_name">
        </div>
        <div class="form-group row">
            <label for="sub" class="col-form-label col-sm-3 text-info label-tooltip" tooltip="Token subject identifier. This is a persistent and immutable identifier for the user that the token describes. Use this value in caching logic.">Subject
                Identifier</label>
            <input type="text" name="sub" class="form-control col-sm-9" disabled [value]="adalService.userInfo.profile.sub">
        </div>
        <div class="form-group row" *ngIf="adalService.userInfo.profile.family_name">
            <label for="family_name" class="col-form-label col-sm-3 text-info label-tooltip" tooltip="User’s last name or surname. The application can display this value.">Last
                Name</label>
            <input type="text" name="family_name" class="form-control col-sm-9" disabled [value]="adalService.userInfo.profile.family_name">
        </div>
        <div class="form-group row" *ngIf="adalService.userInfo.profile.given_name">
            <label for="given_name" class="col-form-label col-sm-3 text-info label-tooltip" tooltip="User’s first name. The application can display this value.">First
                Name</label>
            <input type="text" name="given_name" class="form-control col-sm-9" disabled [value]="adalService.userInfo.profile.given_name">
        </div>
        <div class="form-group row">
            <label for="nonce" class="col-form-label col-sm-3 text-info">One Use Security Key</label>
            <input type="text" name="nonce" class="form-control col-sm-9" disabled [value]="adalService.userInfo.profile.nonce">
        </div>
        <div class="form-group row" *ngIf="adalService.userInfo.profile.pwd_exp">
            <label for="pwd_exp" class="col-form-label col-sm-3 text-info">Password Expires</label>
            <input type="text" name="pwd_exp" class="form-control col-sm-9" disabled [value]="adalService.userInfo.profile.pwd_exp">
        </div>
        <div class="form-group row" *ngIf="adalService.userInfo.profile.pwd_url">
            <label for="pwd_url" class="col-form-label col-sm-3 text-info">Password Change URL</label>
            <input type="text" name="pwd_url" class="form-control col-sm-9" disabled [value]="adalService.userInfo.profile.pwd_url">
        </div>
        <div class="form-group row">
            <label for="aud" class="col-form-label col-sm-3 text-info label-tooltip" tooltip="Raw header options with token that can be cut and pasted into tools such as Postman or Fiddler to test secured API endpoints.">Headers</label>
            <textarea rows="20" class="form-control col-sm-9" disabled>{{ 'Content-Type: application/json\nAuthorization: Bearer ' + adalService.userInfo.token }}</textarea>
        </div>
    </div>
</div>
```

3. Add the following code to the file `src/app/components/token.component.scss`.

```css
/* set class and add a tooltip attribute to show tooltip popup */
.label-tooltip:hover:after {
  color: white;
  content: attr(tooltip);
  position: absolute;
  background: black;
  padding: 2px;
  border-radius: 3px;
}
```

3. Add the following code to the file `src/app/components/token.component.ts`.

```js
import { Component, OnInit } from '@angular/core';
import { Router } from '@angular/router';
import { Location } from '@angular/common';

import { AdalService } from 'adal-angular4';
import { AppInsightsService } from '../../services/app-insights.service';

@Component({
  selector: 'app-token',
  styleUrls: ['./token.component.css'],
  templateUrl: './token.component.html'
})
export class TokenComponent implements OnInit {
  constructor(
    private _location: Location,
    private _router: Router,
    public adalService: AdalService,
    private appInsightsService: AppInsightsService
  ) {}

  ngOnInit(): void {
    this.appInsightsService.logPageView('token.component', '/token');
  }

  getDateString(num: number): string {
    let returnString = '';
    if (num) {
      returnString = num + ' (' + new Date(num * 1000) + ')';
    }
    return returnString;
  }

  logout(): void {
    this.adalService.logOut();
  }
}
```

4. Add the new component `TokenComponent` to the `declarations` section of the file `src/app/app.module.ts`

6. Add the new component to `Routes[]` variable in the file `src/app/app-routing.module.ts`, making it a routable component

```js
{ path: 'token', component: TokenComponent, canActivate: [AdalGuard] }
```

## Create Navigation Menu Component

1. Create folder `srs/app/components/nav-top` containing the files `nav-top.component.html`, `nav-top.component.scss`, and `nav-top.component.ts`.

2. Add the following code to the file `src/app/components/nav-top.component.html`.  Replace `<appname>` appropriatly, and also replace `<modulename>` and `<modulepath>` with the first module created fromt he previous section.

```js
<nav class="navbar navbar-dark bg-dark navbar-expand-md fixed-top">
  <a class="navbar-brand" [routerLink]="['/']"><appname></a>
  <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarSupportedContent"
    aria-controls="navbarSupportedContent" aria-expanded="false" aria-label="Toggle navigation">
    <span class="navbar-toggler-icon"></span>
  </button>
  <div class="collapse navbar-collapse" id="navbarSupportedContent">
    <ul class="navbar-nav mr-auto">
      <li class="nav-item" routerLinkActive="active">
        <a class="nav-link" [routerLink]="['<modulepath>']"><modulename></a>
      </li>
      <li *ngIf="authenticated" class="nav-item" routerLinkActive="active">
        <a class="nav-link" [routerLink]="['token']">Token</a>
      </li>
    </ul>
    <div *ngIf="authenticated" class="nav navbar-right hidden-xs hidden-sm text-light">
      {{ adalService.userInfo.profile.given_name + ' ' + adalService.userInfo.profile.family_name }}
    </div>
    &nbsp;&nbsp;
    <span *ngIf='isConnected' class='online'>ONLINE</span>
    <span *ngIf='!isConnected' class='offline'>OFFLINE</span>
    &nbsp;&nbsp;
    <form class="navbar-form navbar-right">
      <input *ngIf="!authenticated" (click)="login()" type="button" class="btn btn-primary" value="Login">
      <input *ngIf="authenticated" (click)="logout()" type="button" class="btn sm btn-secondary" value="Logout">
    </form>
  </div>
</nav>
```

3. Add the following code to the file `src/app/components/nav-top.component.scss`.

```css
.offline {
    color: red;
}
.online {
    color: greenyellow;
}
```

4. Add the following code to the file `src/app/components/nav-top.component.ts`.

```js
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Router } from '@angular/router';
import { Subscription } from 'rxjs';

import { ConnectivityService } from 'angular-connectivity'; // My NPM Package
import { AdalService } from 'adal-angular4';

@Component({
  selector: 'app-nav-top',
  styleUrls: ['./nav-top.component.scss'],
  templateUrl: './nav-top.component.html'
})
export class NavTopComponent implements OnInit, OnDestroy {
    shrinkNavbar = false;
    isConnected = true;
    subscriptions: Subscription[] = [];
    width =  window.innerWidth;

  constructor(
    public adalService: AdalService,
    private connectivityService: ConnectivityService,
    private router: Router
  ) {}

  onScroll(event: any): void {
    // Shrink the header top and bottom padding when scrolling beyond 300px
    this.shrinkNavbar =
      (window.pageYOffset || document.documentElement.scrollTop) > 300;
  }

  ngOnInit(): void {
    this.adalService.handleWindowCallback();
    const url = localStorage.getItem('url');
    if (url != null) {
        localStorage.removeItem('url');
        this.router.navigateByUrl(url);
    }
    this.subscriptions.push(this.connectivityService.isConnected$.subscribe(isConnected => this.isConnected = isConnected));
    window.onresize = () => this.width = window.innerWidth;
  }

  ngOnDestroy(): void {
    // Unsubscribe all subscriptions to avoid memory leak
    this.subscriptions.forEach(subscription => subscription.unsubscribe());
  }

  login(): void {
    localStorage.setItem('url', this.router.url);
    this.adalService.login();
  }

  logout(): void {
    this.adalService.logOut();
  }

  get authenticated(): boolean {
    return this.adalService.userInfo.authenticated;
  }
}
```

5. Add the new component `NavTopComponent` to the `declarations` section of the file `src/app/app.module.ts`

6. Add the new component to the file `src/app/components/app/app.component.html` above the `router-outlet` like shown below.  This component doies not need to be added to the `app-routing-module.ts` because it is always visible being part of the root `app.component`

```html
<div class="container-fluid">
    <header>
        <app-nav-top></app-nav-top>
    </header>
    <main>
        <router-outlet></router-outlet>
    </main>
</div>
```

## Next Steps

  * `README.md` should be updated appropriately
  * `package.json` should be updated with `description`, `homepage`, `repository.type`, and `repository.url`
  * `favicon.ico` should be replaced with an application specific icon
    * Similarly, all `src/assets/icons` should be replaced with an application specific icons
