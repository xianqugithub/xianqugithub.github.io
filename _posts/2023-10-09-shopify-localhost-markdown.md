---
layout: post
title: Use Localhost Instead of Security Tunnel for Shopify App Development
subtitle: Develop Shopify App WITHOUT Ngrok/Cloudflare
tags: [shopify, hack]
comments: true
---

{: .box-success}
While Shopify has provided a comprehensive framework that allows developers quickly develop apps using Shopify CLI, it is still not light-weighted in the sense that it has hard dependency on network security tunnels such as Ngrok (which is a long-term partner with them), which increases package building time and slows down network communications. It is therefore a frequently asked question to know how to by-pass such requirement. In this article, I will talk about technical hacks we used to develop a full-stack Shopify app without using any security tunnel.

## Assumption
The following steps are largely based on the assumption that the project is generated using Shopify CLI where a full-stack project is contained in a mono-repo that includes both the front and back ends where the back-end synchronously reads a static file from front-end folder in a server-side rendering manner. The back-end is powered by the Express framework in NodeJS and the front-end is developed with React with Vite as the building system. An example project can be found in Shopify’s official Github repository.

## General Idea
The general idea is to trick ***`@shopify/shopify-app-express`*** to believe its deployed in a non-localhost environment with TLS enabled. Thus we can:
1. <span style="color:green">Alter the DNS entry in the operating system to map the fake domain to localhost.</span>
2. <span style="color:blue">Use local SSL proxy to redirect the traffic from HTTPs to HTTP for request processing.</span>

![ShopifyLocalTestSequence](https://xianqugithub.github.io/assets/img/shopify-localhost-sequence.jpeg){: .mx-auto.d-block :}

## Step by Step Procedure

### Global Local Environment Setup

1. In your local **`/etc/hosts`** file, add a new line at the bottom:
   ```console
   127.0.0.1 www.fake-shopifytest.com
   ```
       
2. Install **local-ssl-proxy**:
   ```console
   npm install --location=global local-ssl-proxy
   ```

3. Set the following environment variables:
    ```console
    export SHOPIFY_API_KEY="Client API KEY"
    export SHOPIFY_API_SECRET="Client Secret"
    export SCOPES=Scopes Required By Your App for OAuth"
    export BACKEND_PORT=3000
    ```

### Shopify App Setup In Partner Portal

1. Change ***App URL*** to be `https://www.fake-shopifytest.com`
2. Change ***Allowed Redirection URLs*** to contain the following entries:
  - `https://www.fake-shopifytest.com/auth/callback`
  - `https://www.fake-shopifytest.com/auth/shopify/callback`
  - `https://www.fake-shopifytest.com/api/auth/callback`

### App Repository Changes

#### Change `vite.config.js` Configuration to Support Hot Module Reloading
In the template app generated by Shopify CLI, `hmrConfig` for non-localhost uses `wss` protocol and `443` for `clientPort`. As now everything is literally going through localhost anyway, change it to the same setting:
```javascript
let hmrConfig;
if (host === "localhost") {
    hmrConfig = {
        protocol: "ws",
        host: "localhost",
        port: 64999,
        clientPort: 64999
    };
} else {
    hmrConfig = {
        protocol: "ws",
        host: "localhost",
        port: 64999,
        clientPort: 64999
    };
}
```

#### Change Hot Module Reloading Configurations 

1. Include **http://localhost:3000** in **`dev_embed.js`**:
    ```javascript
    import RefreshRuntime from "http://localhost:3000/@react-refresh";
    ```

2. Change **src** entry in **`index.html`** as follows:
    ```
    <!--Note that for production build(e.g. vite build) this needs be changed back so that the path can be resolved properly.-->
    <script type="module" src="http://localhost:3000/src/index.jsx"></script>
    ```

### Start Hacking

1. Install dependencies:
   ```console
   npm install
   ```

2. Start proxy in a dedicated terminal:
   ```console
   sudo local-ssl-proxy --source 443 --target 3000
   ```

3. Run developmental back-end server in a dedicated terminal:
   ```console
   cross-env NODE_ENV=development HOST=https://www.fake-doppio-shopifytest.com NODE_TLS_REJECT_UNAUTHORIZED=0 nodemon index.js
   ```

4. Run developmental front-end server in a dedicated terminal:
   ```console
   vite --port 3000
   ```

Use this URL to install the app on the test store in the browser: <https://www.fake-shopifytest.com/?shop={your-test-store}.myshopify.com>. If everything is setting up properly, you should be redirected to app installation page. From there you can start making changes and see live updates of the app. Happy hacking. 


