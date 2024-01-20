---
layout: post
title: Develop Customized Shopify Apps for Multiple Stores 
subtitle: How to Develop NodeJS Middlewares to Break Circular Dependency
tags: [shopify, hack, nodejs, middleware]
comments: true
---
{: .box-success}
**PM**: Can we pilot this Shopify App on a few customers before we formally submit our application that can take several weeks?\
\
**SDE**: That's a good idea. How many customers do you want to pilot on?\
\
**PM**: Well, it depends. The leadership hasn't decided yet, and we are subject to many other stake holders such as legal and finance. It can be as low as 1 or as many as 50, so your product should be **FLEXIBLE** enough to deploy as many as we want. And also remember, speed matters in business, so once the decision is made, the deployment must be fast so that we can start the pilot as soon as possible. \
\
**SDE**: That is technically contradicts with Shopify's technical limitation to only allow the customized app to be installed on a single customer. So without that approval of the application, we are encountering a **CIRCULAR DEPENDENCY**. \
\
**PM**: You are the engineer and should be creative. I gotta go to another leadership meeting now. Let's sync next week about the progress.\
\
This is a conversation that literally happened before (of course with some tweaks). But it does illustrate some common scenarios during product development where developers must be really creative to bypass hurdles. In this article, we would like to share a few key techniques that allow **CUSTOMIZED** Shopify Apps to be installed multiple stores. 

## Problem Dissection 
After carefully studying Shopify App development source code, we realized that the key constraint here is the **OAuth** of Shopify. So the problem can be translated to: ***Can we decode the request and re-authenticate the request with shop specific token?*** As Shopify use OAuth and JWT token, we are able to decode the JWT token from HTTP Bearer, so we are all set for the first issue. How about re-authentication part? Remember that in Shopify's NodeJS framework, the authentication is carried out using middlewares, so as long as there is an in-memory cache where the corresponding credentials can be acquired, app specific authentication then is achievable. And of course, everything can be implemented using NodeJS middleware pattern to avoid boiler plate.

The architectural requirement can thus be summarized as: 

![ShopifyMiddleWare](http://xianqugithub.github.io/assets/img/shopify-middleware.jpeg)

## Build Script
This script is triggered during docker building process. The key ingredient here is the `SHOPIFY_STORES` variable that contains the shop specific credential information. In the example here, the information is stored in AWS Secret Manager and retrieved at run time for security purpose. 
    
```
#!/bin/bash
echo "Retrieving Configuration Files and Building"
sc=$(aws secretsmanager get-secret-value \
         --secret-id ShopifySecret \
         --region us-east-1 | jq -r '.SecretString|fromjson')
echo "Finished Retriving Configs"

echo "Building Front End"

# This is provides store specific credential mapping.
export SHOPIFY_STORES=$(jq -r '.SHOPIFY_STORES' <<< "$sc")

# This is for default app credential.
export SHOPIFY_API_KEY=$(jq -r '.SHOPIFY_API_KEY' <<< "$sc")
export SHOPIFY_API_SECRET=$(jq -r '.SHOPIFY_API_SECRET' <<< "$sc")

# Frontend will use vite to build
# At this timepoint, all environment variables will be available for vite to refer
npm run build-frontend

echo "Finished Building Front End for Default"

echo "Backend Preparing to Serve Requests..."
npm run serve
echo "Backend Now Serving Requests."

echo "App Build And Serve Script Complete."

```
    


## Back-end 
### Top-level App Configuration
On the application level, the express instance should be configured to use the correct set of credentials to process the request. In the context of Shopify framework, that task can be delegated to a distinct instance of **@shopify/shopify-app-express** app.

```
// Process request in shop specific manner
app.use(selectShopMiddleware());

// OAuth Configuration
app.get(
    '/api/auth',
    useShopifyApp((shopifyApp) => shopifyApp.auth.begin())
);

app.get(
    '/api/auth/callback',
    useShopifyApp((shopifyApp) => shopifyApp.auth.callback()),
    useShopifyApp((shopifyApp) => shopifyApp.redirectToShopifyOrAppRoot())
);

```

### Middlewares

#### Select Shop Middleware
This middleware is the first layer to process request. Because it can encounter a new shop, it is responsible for return cached instance or create a new instance of Shopify App.

```
function selectShopMiddleware() {
    return (req, res, next) => {
        const shop = extractShopFromReq(req);

        if (shop && !res.locals?.shopify?.app) {
            let shopifyApp = shopifyAppMap[shop];

            if (!shopifyApp) {
                if (hasShopCredential(shop)) {
                    shopifyApp = createShopifyApp(shop);
                } else {
                    shopifyApp = defaultShopifyApp;
                }

                shopifyAppMap[shop] = shopifyApp;
            }

            res.locals.shopify = {
                ...res.locals.shopify,
                app: shopifyApp,
            };
        }

        next();
    };
}
```


#### Use Shopify App Middleware
Because all instance creation should have already been finished by the previous middleware, this middleware only reads from the cache to find the existing instance of Shopify App. 

```
function useShopifyApp(useShopifyAppHandler) {
    return (req, res, next) => {
        const shopifyApp = getShopifyApp(req, res);
        const handler = useShopifyAppHandler(shopifyApp);
        if (Array.isArray(handler)) {
            // This can be buggy, haven't tested it yet
            handler.forEach((h) => {
                h(req, res, next);
            });
        } else {
            handler(req, res, next);
        }
    };
}
```


#### Extract Session Shop
This is a pure HTTP request processing function that extract information using Shopify specific header with the helper of JWT decoding.
```
function extractShopFromReq(req) {
    if (req.get('x-shopify-shop-domain')) {
        return req.get('x-shopify-shop-domain');
    }

    if (req.query?.shop) {
        return req.query.shop;
    }

    let sessionShop = '';
    const authHeader = req.get('authorization');
    if (authHeader) {
        const matches = authHeader.match(/^Bearer (.+)$/);
        if (!matches) {
            logMessage('No match found for Authorization Header');
            return;
        }
        try {
            const jwtPayload = decodeJWT(matches[1]);
            sessionShop = jwtPayload.dest.replace(/^https:\/\//, '');
        } catch (e) {
            logError('Unable to decode JWT', e);
        }
    }

    return sessionShop;
}


function decodeJWT(token) {
    const base64Url = token.split('.')[1];
    const base64 = base64Url.replace(/-/g, '+').replace(/_/g, '/');
    const jsonPayload = JSON.parse(
        Buffer.from(base64, 'base64').toString('binary')
    );
    return jsonPayload;
}
```


#### Get Cached Instance of Shopify App
This is a pure look up function that use extracted shop to lookup the corresponding information in in-memory cache or express response object that put by **selectShopMiddleware**.
```
function getShopifyApp(req, res) {
    const shop = extractShopFromReq(req);

    if (shopifyAppMap[shop]) {
        return shopifyAppMap[shop];
    }

    if (res?.locals?.shopify?.app) {
        return res.locals.shopify.app;
    }

    logError(
        `Cannot find store for ${shop}, using default shopify app. This can be problematic.`
    );

    return defaultShopifyApp;
}
```

## Front-end 

### Vite Configuration
As Store specific credentials are required by AppBridge for authenticated requests, credentials retrieved from Secret Manager will be needed to be referred.

```
export default defineConfig({
    # ... Other configuration code omitted
    define: {
        "process.env.SHOPIFY_API_KEY": JSON.stringify(process.env.SHOPIFY_API_KEY),
        "process.env.SHOPIFY_STORES": process.env.SHOPIFY_STORES
    },
})
```

### AppBridge Provider Configuration
Now because the credentials have already been passed to front-end during build time, in order to let **AppBridge** to find the right configuration, we need to change the configuration to dynamically bind to the shop. The shop information can be accessed from URL, thus we can make a change that uses the shop as the key to look up credentials instead of using static credentials. Here is an example of using the **useState** React hook:


```
const [appBridgeConfig] = useState(() => {
        const params = new URLSearchParams(location.search);
        const host = params.get("host") || window.__SHOPIFY_DEV_HOST;
        
        {/* Extract the shop parameter from the URL that will be used as the key for credential lookup. */}
        const shop = params.get("shop");

        window.__SHOPIFY_DEV_HOST = host;

        {/* This comes from vite configuration mentioned in the previous section. */}
        const storeCreds = process.env.SHOPIFY_STORES;

        let apiKey;

        {/* Iterate through all entries to find the match store's credential. */}
        if (storeCreds) {
            Object.entries(storeCreds).forEach((e) => {
                const [k, v] = e;

                if (shop === k) {
                    apiKey = v.SHOPIFY_API_KEY;
                }
            });
        }

        {/* Use default api key, if credential not found for a given shop. */}
        if (!apiKey) {
            apiKey = process.env.SHOPIFY_API_KEY;
        }

        return {
            host,
            apiKey,
            forceRedirect: true
        };
    });
```
