# Creating Reusable NPM Packages

This document will discuss how to create, publish and re-use NPM packges in your Angular projects.  When an Angular service, pipe, controller or similar item has future use possibilities for other applications, it is best to develop separatly and then import it into your application like any other 3rd party NPM package.  For this exercise, we will walk through the steps to create the re-usable packages that will be later imported into the main angular-template website.

## Steps

1. Create a new angular application that will be dedicated to development and testing of re-usable libraries.

```cmd
ng new angular-libraries
```

2. From the application's root folder, generate a new library within this application using the following code example:

```cmd
ng generate library angular-connectivity
```

* This will create a project within the parent application's folder structure containing its own ```package.json```, ```tsconfig```, and other supporting files and folders needed.

3. From the project's src/lib folder, remove any created files that are not needed, and add of edit the files that will make up this library.  For this example, we will keep only one service file named ```connectivity.service.ts``` with the following content:

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

4. If the service, pipe, etc. being developed has any dependencies, make sure to add them to the libraries ```package.json``` file and not the application's root ```package.json``` file.

5. Edit the file named ```public-api.ts``` located in the project's ```src``` folder.  This file should only include items to be exported by your library.  In our case, this would be a single export for the connectivity service:

```js
/*
 * Public API Surface of angular-connectivity
 */

export * from './lib/connectivity.service';
```

6. Edit the libraries ```README.md``` file to accuratly reflect what this library does and how it should be installed and used.  This information will later be displayed on the NPM registry once the library has been published.

7. The main application can now be used for testing/debugging the libraries functionality by importing it into ```app.module.ts``` like you would any other angular component. 

```js
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';

import { AppComponent } from './app.component';

import { ConnectivityService } from '../../projects/angular-connectivity/src/public-api';

@NgModule({
  declarations: [
    AppComponent
  ],
  imports: [
    BrowserModule
  ],
  providers: [ConnectivityService],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

8. The ```app.component``` can now be used for testing any planned functionality.  Example:

```html
<!--app.component.html-->
<div class="root">
    <span *ngIf='isConnected' class='online'>ONLINE</span>
    <span *ngIf='!isConnected' class='offline'>OFFLINE</span>
</div>
```

```js
// app.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';

import { ConnectivityService } from '../../projects/angular-connectivity/src/public-api';
import { Subscription } from 'rxjs';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.scss']
})
export class AppComponent implements OnInit, OnDestroy {
    isConnected = true;
    subscriptions: Subscription[] = [];

    constructor(private connectivityService: ConnectivityService) {}

    ngOnInit(): void {
        this.subscriptions.push(this.connectivityService.isConnected$.subscribe(isConnected => this.isConnected = isConnected));
    }

    ngOnDestroy(): void {
        // Unsubscribe all subscriptions to avoid memory leak
        this.subscriptions.forEach(subscription => subscription.unsubscribe());
    }
}
```

9. Once satisfied the library is ready, publish it to NPM.  You will need to have an account on [https://www.npmjs.com/](https://www.npmjs.com/).  packages can be published as public or private, and managed by an organization or an individual.

```cmd
ng build angular-connectivity --prod
cd dist/angular-connectivity
npm login
npm publish
```

10. At this point, any other development efforts can include the new library into their codebase using the smae ```npm install``` methods as with any 3rd party packages.

The main application can contain any numbner of libraries with each of them still being publishable as separate NPM packages.  If another library is needed, just follow this process again starting from step #2 above.  All the code shown in this document along with another library containing 2 Angular pipes for sorting and filtering can be found here [https://github.com/PaulGilchrist/angular-libraries](https://github.com/PaulGilchrist/angular-libraries)

For a more detailed walk-through, follow this guide [Building and publishing Angular libraries using Angular CLI](https://medium.com/@faxemaxee/building-and-publishing-angular-libraries-using-angular-cli-140057d21101)