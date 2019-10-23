# Angular Service Workers - Progressive Web Applications (PWA)

Progressive Web Applications (PWA) allow a web application to behave more like a native application.  PWA support caches the entire application locally, allowing the application to run even when the network is unavailable, as well as improving startup performance.  Angular nativly supports PWA using service workers.  Angular maintains the application manifest and downloads the full application in the background after the home page is ready, and when changes exist, switches to the new version at next startup.

Progressive Web Applications (PWA) and Service Workers are not specific to Angular and supported across all modern major browsers, but for the purpose of this document, we will only be discussing how to implement this capability into an Angular application.

## Steps

1. Add a Service Worker to the project

```cmd
ng add @angular/pwa --project <project-name>
```

2. Service Workers do not support ```ng serve``` and must be tested using a production build and an HTTP server.  If you do not have an http server globally installed, it is recommended to install the following one shown below:

```cmd
npm install -g http-server
```

3. Build and run the production project for testing the new Service Worker capabilities

    * ```<project-name>``` is only needed if your project is in a subfolder under ```dist/```

```cmd
ng build --prod
http-server -p 8080 -c-1 dist/<project-name>
```

4. Launch the browser, open its debugging tools, and go to the network tab.  Entering the website's URL, while watching the network tab should show the Service Worker loading all the applicatiuon's files locally.

5. After the initial load is complete, refreshing the application using ```F5``` should show the application now reloading from the Service Worker local files rather than the network.  For this process, the Service Workers looks at the application manifest to determine if any files have changed.  A file will be served from its local copy anytime that file has not changed, or the network is unavailable.

6. The network tab should have an option for switching between online and offline.  Changing this setting to offline and refreshing the application should show the application continuing to function with the network unavailable.

It is up to the application to determine how to handle remote API calls or access to other remote systems when the network is offline.  The browser exposes a variable named ```window.navigator.onLine``` for the status of network connectivity.  It is best to wrap this variable in an event listner to be notified whenever its status changes.  The following is a simple service to wrap this variable in a BehaviorSubject for easier event notification.  This server is available as an NPM package named ```angular-connectivity```

```js
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';

@Injectable()
export class ConnectivityService {

    private isConnected = new BehaviorSubject<boolean>(!!window.navigator.onLine);
    isConnected$ = this.isConnected.asObservable();

    constructor() {
        window.addEventListener('online', () => {
            this.isConnected.next(!!window.navigator.onLine);
            console.log('Internet connection established');
        });
        window.addEventListener('offline', () => {
            this.isConnected.next(!!window.navigator.onLine);
            console.log('Internet connection lost');
        });
    }

}
```

Leveraging the ```angular-connectivity``` in combination with conditionals like *ngIf allows for making UI or logic decisions based on connectivity.  For example, certian menu items can be hidden or disabled when network connectivity is not available.  Similarly, data changes could be saved locally when connectivity is not available, and later sent to a remote API when network connectivity is re-established.  Both of these examples can be seen in the demo application named ```angular-template``` located here [https://github.com/PaulGilchrist/angular-template](https://github.com/PaulGilchrist/angular-template)


## References

* [Service Workers - Practical Guided Introduction (several examples)](https://blog.angular-university.io/service-workers/)
* [Getting Started with Service Workers](https://angular.io/guide/service-worker-getting-started)
* [Angular Service Worker - Step-By-Step Guide for turning your Application into a PWA](https://blog.angular-university.io/angular-service-worker/)
* [Angular Service Worker Introduction](https://angular.io/guide/service-worker-intro)
* [Service Worker Communication](https://angular.io/guide/service-worker-communications)
* [Service Worker in Production](https://angular.io/guide/service-worker-devops)