# Handling Server API Throttling and Paging

In order to prevent a single requestor from consuming an unpreportionate amount of resources, server side APOs frequently enforce throttling and/or paging.  This document will discuss how an Angular application can handle these responses ensuring the data requested is successfully retrieved.

## Paging

When an OData API server limits the number of returned objects below that requested, its response body object will contain an additional property named `@odata.nextLink`.  You can use the GitHub project [api-template-odata-core-framework](https://github.com/PaulGilchrist/api-template-odata-core-framework) to show this behavior and test against.

The following Angular example code shows how to assemble the pages individual responses (pages) until all the data is returned:

```js
users = [];

ngOnInit() {
    this.getUsers('<your API url here>');
}

getUsers(url) {
    this.http.get(url)
        .pipe(
            tap((data: any) => {
                this.users = this.users.concat(data.value);
                const nextLink = data['@odata.nextLink'];
                if (nextLink) {
                    this.getUsers(nextLink);
                } else {
                    //  We now have all the users regardless of what paging limit was set to on the server
                    console.log(this.users);
                }
            })
        ).subscribe();
}
```

This example does not bother with error handling, retry logic, or behaviorSubjects in order to keep the example as simple and focused as possible.  It should be even easier if you were doing it API to API, since async and await could make this a simple while loop, but that will not work if you decided to implement your own paging limits in the requesting application’s API.

## Throttling

An API will respond with an HTTP error code 429 (Too Many Requests) when needing to throttle a requestor.  When this response is received, the requestor should delay for a short period of time and then retry the request.  Angular has an `http pipe` named `retryWhen` that allows you to pass your own retry function.  The following example code allows for controlling the number of times to retry, the rate at which to increase the delay between retrys, and what status codes that if returned should not be retired.

```js
export const genericRetryStrategy = (
        { maxRetryAttempts = 3, scalingDuration = 1000, excludedStatusCodes = [] }
        :
        { maxRetryAttempts?: number, scalingDuration?: number, excludedStatusCodes?: number[] }
        = {}
    ) => (attempts: Observable<any>) => {
    return attempts.pipe(
        mergeMap((error, i) => {
            const retryAttempt = i + 1;
            // if maximum number of retries have been met or response is a status code we don't wish to retry, throw error
            if (retryAttempt > maxRetryAttempts || excludedStatusCodes.find(e => e === error.status)) {
                return throwError(error);
            }
            console.log(`Attempt ${retryAttempt}: retrying in ${retryAttempt * scalingDuration}ms`);
            // retry after 1s, 2s, etc...
            return timer(retryAttempt * scalingDuration);
        }),
        finalize(() => console.log('We are done!'))
    );
};
```

Next you would simply add “retryWhen(genericRetryStrategy())” to the existing http pipe like in the below example:

```js
getUsers(url) {
    this.http.get(url)
        .pipe(
            retryWhen(genericRetryStrategy()),
            tap((data: any) => {
                this.users = this.users.concat(data.value);
                // Handle any possible server side page limits
                const nextLink = data['@odata.nextLink'];
                if (nextLink) {
                    this.getUsers(nextLink);
                } else {
                    //  We now have all the users regardless of what paging limit was set to on the server
                    console.log(this.users);
                }
            })
        ).subscribe();
}
```

Notice that this is the same example used for `paging` above with just the addition of retry logic. For testing only, to force a 429 error before calling “getUsers()” you could force hitting the limit first like this:

```js
ngOnInit() {
    const url = '<your API url here>'
    // Lets force an error 429 (Too Many Requests)
    for (let i = 0; i < 5; i++) {
        this.http.get(url).subscribe();
    }
    // Now that the server has throttled us, lets make a call to show we can handle the throttling properly
    this.getUsers(url);
}
```

This testing code assumed your test API server is throttling at a low enough rate for this small loop to induce the 429 response (4 requests per 5 seconds for example).  Either adjust your test APIs throttling policy, or the above Angular code appropriatly to induce the 429 error response, then call your `http.get()` request with the retryWnen() logic to test its behavior.
