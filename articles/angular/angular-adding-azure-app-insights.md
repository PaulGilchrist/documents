# Angular - Adding Azure Application Insights

Azure Application Insights allows for collecting, reporting, and alerting on both server and client side telemetry.  Common client side telemetry includes page views, events, exceptions, and trace data.  The following steps will add the ability for a new or existing Angular application to send Application Insights telemetry to Azure:

1. In the `package.json` file, add `applicationinsights-js` to the dependencies section and `@types/applicationinsights-js` to the devDependencies section, choosing their latest versions, followed by running `npm install` from the applications root folder to install both packages.

2. Add the following section to the `environment` object within each environment file replacing the `instrumentationKey` with the one obtained from Azure.

```json
    applicationInsights: {
        instrumentationKey: 'xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
    },
```

3. In the app module, create a new service named `app-insights.service.ts` with the following code:
   * If you are not using `adal-angular4` for authentication, adjust the code accordingly.

```javascript
import { Injectable } from '@angular/core';
import { AppInsights } from 'applicationinsights-js';

import { environment } from '../../environments/environment';
import { AdalService } from 'adal-angular4';

@Injectable()
export class AppInsightsService {

    private config: Microsoft.ApplicationInsights.IConfig = {
        instrumentationKey: environment.applicationInsights.instrumentationKey
    };

    constructor(public adalService: AdalService) {
        if (!AppInsights.config) {
            AppInsights.downloadAndSetup(this.config);
        }
    }

    logPageView(name?: string, url?: string, properties?: any, measurements?: any, duration?: number) {
        this.setUser();
        AppInsights.trackPageView(name, url, properties, measurements, duration);
    }

    logEvent(name: string, properties?: any, measurements?: any) {
        this.setUser();
        AppInsights.trackEvent(name, properties, measurements);
    }

    logException(exception: Error, handledAt?: string, properties?: any, measurements?: any) {
        this.setUser();
        AppInsights.trackException(exception, handledAt, properties, measurements);
    }

    logTrace(message: string, properties?: any, severityLevel?: any) {
        this.setUser();
        AppInsights.trackTrace(message, properties, severityLevel);
    }

    setUser() {
        if(this.adalService.userInfo.authenticated) {
            AppInsights.setAuthenticatedUserContext(this.adalService.userInfo.profile.upn);
        } else {
            AppInsights.clearAuthenticatedUserContext();
        }
    }

}
```

4. In the app module file, import the newly created service and add it to the `providers` section

5. In any component where you want to send telemetry, include the App Insights service and call the appropriate function matching the type of telemetry to send.  Example below:

```javascript
import { AppInsightsService } from '../../services/app-insights.service';
```

```javascript
constructor(private appInsightsService: AppInsightsService) {}
```

```javascript
ngOnInit() {
    // Example of how to add Application Insights tracking to a component
    this.appInsightsService.logPageView('home.component', '/home');
}
```
