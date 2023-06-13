---
layout: post
title: "Analysis for Failed to fetch CSRF token in @sap-cloud-sdk/http-client"
# subtitle: "This is the post subtitle."
comments: true
date:   2023-06-13 12:10:21 +0800
categories: 'en'
---

## TL;DR

1. HEAD is not automatically supported for cds odata action/function.
2. APIM policy may configured to only accept `GET` and `POST`

## Issue Description

We received several warning logs in prod environment which showed that lots of csrf token request failed.

## How to reproduce

1. Go to dev APIM and Click 'Test' in the left tab, then start debug as below:

   ![image-20230612135640689](/assets/images/posts/2023-06-13/image-20230612135640689.png)

2. Clear user cache (take my i-number as an example): `hdel user zzz`

3. Verify user cache is empty: `hget user zzz`, Should return 'No result'

4. Enter: <https://origin.learning-dev.sap.com/service/runtime/les/getUserEntitlement()>

   ![image-20230612135821245](/assets/images/posts/2023-06-13/image-20230612135821245.png)

5. Go back and click Refresh, two error request with HEAD, one is '/userLogin/' and another is '/userLogin':

   ![image-20230612135932088](/assets/images/posts/2023-06-13/image-20230612135932088.png)

## Further analysis

1. In dev environment, open kibana and search with `"userLogin" AND component_name:"progress" AND "HEAD" AND organization_name:"sap-learning-dev-venusaur"`

    ![image-20230612152957380](/assets/images/posts/2023-06-13/image-20230612152957380.png)

2. Copy correlation_id for these two requests: `/userLogin` and `/userLogin/`

    ![image-20230612153124617](/assets/images/posts/2023-06-13/image-20230612153124617.png)

    ![image-20230612153145501](/assets/images/posts/2023-06-13/image-20230612153145501.png)

3. We can see two HEAD requests sent to progress application and both failed.

4. Then we checked the source code and found that every time the error thrown, there will be a debug level log indicating that fetch csrf token failed.
  
    - BTW, in data-approuter, the @sap-cloud-sdk/http-client is "^2.7.1" and in runtime it's "^3.1.1", the csrf request implementation is slightly different:

      ```javascript
      // in 2.7.1
      // csrf-token-header.js
      function makeCsrfRequest(destination, requestConfig) {
          const axiosConfig = {
              method: 'head',
              ...requestConfig,
              params: {
                  custom: requestConfig.params || {},
                  requestConfig: {}
              },
              headers: {
                  custom: buildCsrfFetchHeaders(requestConfig.headers),
                  requestConfig: {}
              }
          };
          // The S/4 does a redirect if the CSRF token is fetched in case the '/' is not in the URL.
          // TODO: remove once https://github.com/axios/axios/issues/3369 is really fixed. Issue is closed but problem stays.
          const requestConfigWithTrailingSlash = appendSlash(axiosConfig);
          return (0, http_client_1.executeHttpRequest)(destination, requestConfigWithTrailingSlash)
              .then(response => response.headers)
              .catch(error1 => {
              const headers1 = getResponseHeadersFromError(error1);
              if (hasCsrfToken(headers1)) {
                  return headers1;
              }
              logger.warn(new util_1.ErrorWithCause(`First attempt to fetch CSRF token failed with the URL: ${requestConfigWithTrailingSlash.url}. Retrying without trailing slash.`, error1));
              const requestConfigWithOutTrailingSlash = removeSlash(axiosConfig);
              return (0, http_client_1.executeHttpRequest)(destination, requestConfigWithOutTrailingSlash)
                  .then(response => response.headers)
                  .catch(error2 => {
                  const headers2 = getResponseHeadersFromError(error2);
                  if (hasCsrfToken(headers2)) {
                      return headers2;
                  }
                  logger.warn(new util_1.ErrorWithCause(`Second attempt to fetch CSRF token failed with the URL: ${requestConfigWithOutTrailingSlash.url}. No CSRF token fetched.`, error2));
                  // todo suggest to disable csrf token handling when the API is implemented
                  return {};
              });
          });
      }
      ```

      ```javascript
      // in 3.1.1
      // csrf-token-middleware.js
      async function makeCsrfRequest(requestConfig, options) {
          try {
              const response = await (0, internal_1.executeWithMiddleware)(options.middleware, {
                  fn: axios_1.default.request,
                  fnArgument: requestConfig,
                  context: options.context
              });
              return findCsrfHeader(response.headers);
          }
          catch (error) {
              if (findCsrfHeader(error.response?.headers)) {
                  return findCsrfHeader(error.response?.headers);
              }
              logger.warn(new util_1.ErrorWithCause(`Failed to get CSRF token from  URL: ${requestConfig.url}.`, error));
          }
      }
      ```

5. Check source code and found that the API is odata function, which means it does not support HEAD method, then it explained why we received 405 error.

6. Is that enough?

## Following Questions

1. Why the code is deployed one month ago and only observed recently?

    > In runtime application, the '/userLogin' API is only invoked by '/les/getUserEntitlement()', and it's being tested since 2023-06-06, check logs below:
    >
    > ![image-20230612133448857](/assets/images/posts/2023-06-13/image-20230612133448857.png)

2. '/userLogin' API is also invoked in data-approuter application, and it's being invoked the same way, why it's not observed before?

    > Log config is set to `error` in data-approuter.
    >
    > ![image-20230612155349552](/assets/images/posts/2023-06-13/image-20230612155349552.png)
    >
    > Actually it also failed when send HEAD request from data-approuter to APIM, we can check that from APIM debug console as previously stated.
    >
    > 1. Start debug (RECOMMEND to do in private dev)
    > 2. Go to: <https://sap-learning-private-dev1-com-sap-learning-data-approuter.cfapps.eu10-004.hana.ondemand.com/?userlogin=true>
    > 3. Click refresh and you will see two HEAD request errors

3. Why observed different logs in prod/stage vs test/dev

    > The APIM policies configuration is different, in stage/prod there is a policy to check if the method is allowed:
    >
    > ![image-20230612160731691](/assets/images/posts/2023-06-13/image-20230612160731691.png)
    > ![image-20230612160746024](/assets/images/posts/2023-06-13/image-20230612160746024.png)
    >
    > It's not configured in dev/test:
    > ![image-20230612160817915](/assets/images/posts/2023-06-13/image-20230612160817915.png)

## Good to Know

- DO NOT perform csrf token fetch for internal invocation
- DO NOT manually copy-paste for cross environment configurations
- DO NOT force global log level in the code
- DO NOT force log level to ERROR, it's not friendly for troubleshooting
