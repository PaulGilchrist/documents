# Angular Template Setup - Step by Step

The following steps can be used to start a new Angular project and ensure you are following current Angular best practices.  Do not use existing Visual Studio Angular project templates as they are almost always several versions out of date, and bring in .Net Core standards and overhead not native to Angular.  If desired, Visual Studio can still be used to develop this application after the template initial setup, however VS Code is a more common Angular development environment.

## Steps

1. Ensure all tools are up to date
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

8. Add description, homepage, repository.type, and repository.url to `package.json`

9. Add the following to `build.scripts` and `test.scripts` in the file `angular.json`

```json
"node_modules/jquery/dist/jquery.js",
"node_modules/bootstrap/dist/js/bootstrap.js",
```

10. Add the following to `build.configurations` in the file `angular.json`

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

11. Add the following to `serve.configurations` in the file `angular.json`

```json
"qa": {
    "browserTarget": "purchase-pro:build:qa"
},
"staging": {
    "browserTarget": "purchase-pro:build:staging"
},
```

12. Add the following `imports` in the file `src/app/app.module.ts`

```js
    BrowserAnimationsModule,
    HttpClientModule,
    ToastrModule.forRoot(),
    ServiceWorkerModule.register('sw-worker.js', { enabled: environment.production }) // Replaced ngsw-worker.js to add ability to open application from notification
```

13. Add the following `providers` in the file `src/app/app.module.ts`

```js
    AdalService,
    AdalGuard,
    AppInsightsService,
    ConnectivityService
```

14. Raise max-line-length to 350 in file `tslint.json`

15. Setup app setting in the folder `environments` the files `environment.ts`, `environment.qa.ts`, `environment.staging.ts`, and `environment.prod.ts` following the below template, but changing the values as needed.

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
        userData: 3600000 // 10 minutes
    },
    envName: 'dev',
    vapid: {
        publicKey: 'BE6IqM0la0Mr7jg5w5vYwPk5gwbypKcpsJrqH-xX3nfLm9BCCrt2EDCyMZH7yZYFDGtGxtCgZqCgnGeuJehndoc'
    },
    production: false
};
```

16. Create folder `src/app/components/app` and move the following files into the new folder
    * `app.component.html`, `app.component.scss`, `app.component.spec.ts`, and `app.component.ts`

17. Create folder `src/app/services` and copy the following files into the new folder (get from template project or create new)
    * `app-insights.service.ts` - This service allows client side custom telemetry logging to Azure App Insights
    * `auth.interceptor.ts` - This service allows automatically adding the current Oauth bearer token to all HTTP request
    * `identity.service.ts` - This service extends AdalAngular.js functionality allowing for easy checks of a user's roles
    * `logging.interceptor.ts` - This service automatically logs any HTTP related errors to the console

18. Add the above two interceptors to the `providers` section of the file `app.module.ts`

```js
{ provide: HTTP_INTERCEPTORS, useClass: LoggingInterceptor, multi: true }, // Time how long each http requests takes
{ provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true } // Add token to all API requests
```

19. Add the following imports to the file `src/styles.scss`

```css
@import '~animate.css/animate.css';
@import '~bootstrap/scss/bootstrap.scss';
@import '~font-awesome/css/font-awesome.css';
@import '~ngx-toastr/toastr-bs4-alert';
```

20. Optional - App the following to the `Application Imports` section of the file `src/polyfills.ts`

21. Add the following code to the `AppComponent` to initialize `adalAngular.js`, force login, and send telemetry to `1`Azure App Insights`

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

## Next Steps

  * `README.md` should be updated appropriately
  * Add the first application module.  Do not add components directly to the `app.module`
    * Configure the `app.module` to route to a sub-module's home page
  * Choose a new `favicon.ico`
  * Replace the application icons in `src/assets/icons` with application specific icons
