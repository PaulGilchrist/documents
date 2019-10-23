# Web Push Notifications

Web push notification allow for realtime communications to a web client application even if that application is not currently running on the device and even if the browser is not active.  The behavior is very similar to how a native application would receive notifications.  These notifications can be sent broadly to many devices, or personalized and targeted to a specific device. Web Push Notifications also allow for communication without requiring a client to supply personal information such as an email address.

Web push notifications are open source, supported by modern/major browsers, and part of the W3C standards.

## Client Steps

1. Ensure your web client application is already configured with Service Worker support.  See [Service Workers - Progressive Web Applications (PWA)](https://github.com/PaulGilchrist/documents/blob/master/articles/angular/angular-service-workers.md)

2. Globally install the npm package web-push and generate your push server's Voluntary Application Server Identification (VAPID) public and private encryption keys, saving the returned JSON for later use.

```cmd
npm install web-push -g
web-push generate-vapid-keys --json
```

3. Store the VAPID public key in the environment files ```environment.ts```, ```environment.prod.ts```, etc. as there should be different keys used for each environment.

```js
export const environment = {
    vapid: {
        publicKey: 'BE6IqM0la0Mr7jg5w5vYwPk5gwbypKcpsJrqH-xX3nfLm9BCCrt2EDCyMZH7yZYFDGtGxtCgZqCgnGeuJehndoc'
    },
    ...
};
```

4. Add a request to send notifications to somewhere within the application such as ```app.component.ts```. This is also a good place to add a subscription to react to any notification clicks.  This notificationClicks handler will only be executed when the application is running.  Handling notification clicks while the application and even browser are not running is discussed in a later step.

```js
import { Component, OnInit } from '@angular/core';
import { SwPush } from '@angular/service-worker';

import { environment } from '../../../environments/environment';

@Component({
  selector: 'app-root',
  styleUrls: ['./app.component.css'],
  templateUrl: './app.component.html'
})
export class AppComponent implements OnInit {
    constructor(private swPush: SwPush) {}

    ngOnInit(): void {
        // If the user has not already answered, ask them if they would like to receive notifications
        this.swPush.requestSubscription({
            serverPublicKey: environment.vapid.publicKey
        })
        .then(subscription => {
            const notificationSubscription = JSON.stringify(subscription);
            /*
            Would normally send this subscription to the server to store in a database here
                but for the simplicity for this example, we will just store it locally and later hard code it into the server.
            */
            localStorage.setItem('notificationSubscription', notificationSubscription);
            console.log('Successfully subscribed to notifications');
            console.log(notificationSubscription);
        })
        .catch(error => {
            if (Notification.permission === 'denied') {
                console.warn('Permission for notifications was denied');
            } else {
                console.error('Unable to subscribe to notifications', error);
            }
         });
        this.swPush.notificationClicks.subscribe(
            ({action, notification}) => {
                // These will only execute if the application is already open
                // To execute when application is closed, add code to ./sw-worker.js instead of here
                switch (action) {
                }
            }
        );
    }

}

```

5. It is often desirable to respond to notification clicks even when the browser or application are not running.  In that scenario, only the Service Worker will be running, requiring the notification click handler to be part of the Service Worker.  This can be done by extending the Service Worker functionality through creating a file named ```sw-worker.js``` in the ```src``` folder of the project.  This file will import the original Service Worker code and then add any new functionality.  Actions will be explained in an upcoming step, but you can see from the below example code that there based on what action the user clicks, the browser will be launched, and go to a specific URL.

```js
importScripts('./ngsw-worker.js');
self.addEventListener('notificationclick', function(event) {
    event.notification.close();
    var action = event.action;
    var notification = event.notification;
    switch (action) {
        case 'explore-application':
            event.waitUntil(clients.openWindow(notification.data.applicationUrl));
            break;
        case 'explore-github':
            event.waitUntil(clients.openWindow(notification.data.githubUrl));
            break;
    }
});
```

6. To ensure the extended Service Worker functionality is compiled and included with the application, edit ```angular.json``` to include ```src/sw-worker.js``` in the ```assets``` section of the build.

7. Change the Javascript file that is being registered in ```app.module.ts``` from ```ngsq-worker.js``` to the new ```sw-worker.js```

8. Build and run the client application and accept the request to receive notifications.  Make sure to use the debugging tools to save the subscription JSON to be used later on the notification server.

   * ```<project-name>``` is only needed if your project is in a subfolder under ```dist/```

```cmd
ng build --prod
http-server -p 8080 -c-1 dist/<project-name>
```

## Server Steps

To keep this walkthrough as simple and on topic as possible, the server will consist of a single file Node JS application that's sole purpose is to receive notification requests from Postman or a similar API client request for testing the notifications adhock.

1. Create a ```server``` folder under ```src```

2. Add a ```package.json``` file to this folder with the two dependencies  ```express``` and ```web-push```

```json
{
  "authors": [
    "Paul Gilchrist"
  ],
  "name": "notification-server",
  "description": "Test notification server",
  "homepage": "",
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": ""
  },
  "version": "0.0.1",
  "scripts": {
    "ng": "ng",
    "start": "node app.js"
  },
  "dependencies": {
    "express": "~4.17.1",
    "web-push": "~3.4.0"
  },
  "devDependencies": {}
}
```

3. If neccessary, modify the .gitignore file on the project's root folder to ignore the ```node-modules``` that will be created when this ```package.json``` installs these NPM packages

```cs
/src/server/node_modules
```

4. Run NPM from within the ```server``` folder to install the two packages

```cmd
npm install
```

5. Add a file named ```server.js``` that will act as a single endpoint API that can send out notifications to the client when called.  Make sure to edit this file placing in the VAPID keys and subscription JSON you generated in the earlier client steps.

```js
var express = require('express'),
    os = require('os'),
    webpush = require('web-push');

var vapidKeys = { // web-push generate-vapid-keys --json
    "publicKey":"BE6IqM0la0Mr7jg5w5vYwPk5gwbypKcpsJrqH-xX3nfLm9BCCrt2EDCyMZH7yZYFDGtGxtCgZqCgnGeuJehndoc",
    "privateKey":"bW1GCNS8dMosgZr3bO155BRuATT8GVbJhPKPy9ZPmoY"
};

webpush.setVapidDetails(
    'mailto:paul.gilchrist@outlook.com',
    vapidKeys.publicKey,
    vapidKeys.privateKey
);

var app = express();

app.route('/api/notifications').post(sendNotification);

function sendNotification(req, res) {
    var allSubscriptions = [ // This would normally come from a database and contain subscriptions from every customer
        {
            "endpoint":"https://fcm.googleapis.com/fcm/send/fIg349CZ4Jg:APA91bHjG2UHvgDgS7PBz_gwGpY0M49rcvDOaHx3URki80j3QeUfy8SyuvfNCPaor5dne7DfjnhVq3vVw2ovXNsG0OLBtL0pR8hCfcQlsQY0Su45qncMZij1BP7Wy64C4RgRXi4a0FOO",
            "expirationTime":null,
            "keys": {
                "p256dh":"BGQ03cZz9Apc1wZT_p6ZoPdkd6JUP07XdB0JeMuZjmYwUIx0IqdU1yimyCzgGHM85-X6X3S728XtJZXv_vxumcA",
                "auth":"PBQmWyz0rov4QqurjQE7RQ"
            }
        }
    ];
    console.log('Total Subscriptions', allSubscriptions.length);
    var notificationPayload = {
        "notification": {
            "title": "Angular Template Notification",
            "body": "Welcome to the Angular Training Template!",
            "icon": "assets/icons/icon-72x72.png", // Relative to the client not the server
            "vibrate": [100, 50, 100],
            "data": { // Put any data you want here and it will be accessable on the client with the notification
                "applicationUrl": "https://angulartemplate.azurewebsites.net",
                "githubUrl": "https://github.com/PaulGilchrist/angular-template"
            },
            "actions": [ // Can also add icon
                {
                    "action": "explore-application",
                    "title": "View the application"
                },
                {
                    "action": "explore-github",
                    "title": "Checkout our GitHub"
                }
            ]
        }
    };
    Promise.all(allSubscriptions.map(function(sub) {
            webpush.sendNotification(sub, JSON.stringify(notificationPayload));
        })
    )
    .then(function() { return res.status(200).json({message: 'Notification sent successfully.'});})
    .catch(function(err) {
        console.error("Error sending notification, reason: ", err);
        res.sendStatus(500);
    });
}

//Use port provided by hosting provider if available
app.set('port', (process.env.PORT || 3000));
//Start the application
app.listen(app.get('port'), function() {
    console.log('App running on host', os.hostname(), 'and port', app.get('port'));
});
```

6. Create an API call for posting to the server using ```Postman```, ```CURL``` or a similar tool.

```http

    POST http://localhost:3000/api/notifications

```

7. Launch the server application from within the '``server``` folder.  It will be running on port 3000 by default.

```cmd
    node server.js
```

8. Run the Postman API call and watch for the notification to appear on the device.  Clicking on one of the notification buttons should launch the browser and open the respective URL.

All code examples discussed in this docuement can be found within the [angular-template application](https://github.com/PaulGilchrist/angular-template) on GitHub

## References

* [Angular Push Notifications: a Complete Step-by-Step Guide](https://blog.angular-university.io/angular-push-notifications/)
* [Push Notifications in ASP.NET Core with Angular](https://www.telerik.com/blogs/push-notifications-in-asp-net-core-with-angular)
* [Web Push Notifications: Timely, Relevant, and Precise](https://developers.google.com/web/fundamentals/push-notifications)
* [Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)
* [Notifications API](https://developer.mozilla.org/en-US/docs/Web/API/Notifications_API)
* [Generic Event Delivery Using HTTP Push](https://tools.ietf.org/html/draft-ietf-webpush-protocol-12)
* [PushAlert SaaS Solution](https://pushalert.co/)