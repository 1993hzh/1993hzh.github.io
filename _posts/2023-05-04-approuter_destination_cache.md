---
layout: post
title: "Destination cache in Approuter and Connectivity"
# subtitle: "This is the post subtitle."
comments: true
date:   2023-05-04 18:31:45 +0800
categories: 'en'
---

## Connectivity in @sap-cloud-sdk

### TL;DR

* Built-in cache mechanism with `useCache` in option
* By default expiration time is 5 mins and no eviction

---
By default, if we use `executeHttpRequest` from `@sap-cloud-sdk/http-client`, the function signature is:

```typescript
function executeHttpRequest<T extends HttpRequestConfig>(destination: DestinationOrFetchOptions, requestConfig: T, options?: HttpRequestOptions): Promise<HttpResponse>;
```
The first arg accepts a `Destination` or `DestinationFetchOptions`, currently in our system, we always use `DestinationFetchOptions`.

Before `http-client` perform request, it will first try to resolve destination by options, the detailed implementation sequential flow looks like below:

![](/assets/images/posts/2023-05-04/1.jpg)


`destination-accessor.getDestination` actually will not call getDestinationFromDestinationService only, in the above diagram it's just simplified for better understanding, here is the detailed code logic:

```typescript
async function getDestination(options) {
    const destination = (0, destination_from_env_1.searchEnvVariablesForDestination)(options) ||
        (await (0, destination_from_registration_1.searchRegisteredDestination)(options)) ||
        (await (0, destination_from_vcap_1.searchServiceBindingForDestination)(options)) ||
        (await (0, destination_from_service_1.getDestinationFromDestinationService)(options));
    return destination;
}
```
`destination-cache` underlying is using `cache` with fixed 5 mins as expiration time, however, please note that the in-memory cache implementation does not support eviction, so it's important to keep in mind not to cache objects unlimitedly, otherwise it may cause OOM.

---
## Approuter in @sap

### TL;DR

* **NO** global cache, only user session level cache 
* No option to enable/disable cache for destination in `xs-app.json`
* Destination Service in PROD has near half request come from approuter

---

`approuter` is using a totally different approach to retrieve destination as above, here is the simplified sequential diagram:

![](/assets/images/posts/2023-05-04/2.jpg)

`destination-token-middleware.getCacheDestination` is getting cached destination from user session, here is the code:

```javascript
function getCachedDestination(req, key, destinationName){
  let destination = req.session && req.session.user && req.session.user.destinations && req.session.user.destinations[key];
  if (destination && ((destination.destinationConfiguration.authentication === 'BasicAuthentication'
      || destination.destinationConfiguration.authentication === 'NoAuthentication') ||
      (destination.expireDate && destination.expireDate - Date.now() > 0))){
    const logger  = req.loggingContext.getLogger('/Destination service');
    logger.info('Destination ' + destinationName + ' has been retrieved from cache');
    return destination;
  } else {
    return null;
  }
}
```


Despite this, there is no other place in the request flow will cache the destination instance currently, which means each request from different user will result in a call to DestinationService, thus we did a check of backtrace in dynatrace, and found that the result proved the assumption:

BackTrace for DestinationService in DEV:

![](/assets/images/posts/2023-05-04/3.jpg)

BackTrace for DestinationService in PROD:

![](/assets/images/posts/2023-05-04/4.jpg)

---
### More findings
Approuter supports integrate with `HTML5 Application Repository`, where we can change routes dynamically in runtime without redeploy approuter to make changes on `xs-app.json` work, the configuration supports global cache and by default cache expiration time is 5 mins. Detailed logic can be found in `dynamic-routing-util`, documentation can be found in `@sap/approuter - README.md`.

For schema of `xs-app.json` we can find from `xs-app-schema.json`.