---
layout: post
title: Use Localhost instead of Security Tunnel for Shopify App Development Testing
subtitle: Develop Shopify App WITHOUT Ngrok/Cloudflare
tags: [shopify, hack]
comments: true
---

{: .box-success}
While Shopify has provided a comprehensive framework that allows developers quickly develop apps using Shopify CLI, it is still not light-weighted in the sense that it has hard dependency on network security tunnels such as Ngrok (which is a long term partner with them), which increases package building time and slows down network communications. It is therefore a frequently asked question to know how to by pass such dependency and/or one of the top feature request to Shopify CLI by developers. In this article, I will talk about technical hacks we used to develop a full-stack Shopify app without using any security tunnel. 

## General Idea
The general idea is to trick @shopify/shopify-app-express to believe its deployed in a non-localhost environment with TLS enabled. Thus we can:

1. Alter the DNS entry in the operating system to map a fake domain to localhost.
2. Use local SSL proxy to redirect the traffic from port 443(HTTPS) to back-end server port for request processing.
3. (Temporarily) Allow unauthorized request via TLS. 


## Step by Step Procedure
The following steps are largely based on the assumption that the project is generated using Shopify CLI where a full-stack project is contained in a mono-repo that contains both the front and back ends where the back-end reads a static file from front-end folder in a server side rendering manner. The back-end uses is powered by the Express framework in NodeJS and the front-end is developed with React that is built by Vite. An example project can be found in Shopify's official Github repository.

### Local Environment Setup

- In your local `/etc/hosts` file, add a new line at the bottom:

```
127.0.0.1 www.fake-shopifytest.com
```

- Install local SSL proxy and make sure it can run. Leave the proxy running in a separate terminal:

```
# One time operation - you may need to install npm first
npm install --location=global local-ssl-proxy 

# Then run this in a separate terminal
sudo local-ssl-proxy --source 443 --target 3000
```

- Set the following environment variables:

```
export SHOPIFY_API_KEY="<Client ID from your shopify app>"
export SHOPIFY_API_SECRET="<Client secret from your shopify app>"
export SCOPES="the scopes required by the app for OAuth"
export BACKEND_PORT=3000
```

### Shopify App Setup

Under your `partners.shopify.com` account page, go to your app, in App setup tab:

- Change "App URL" to be `https://www.fake-shopifytest.com`
- Change "Allowed redirection URL(s)" to contain the following entries:
  - `https://www.fake-shopifytest.com/auth/callback`
  - `https://www.fake-shopifytest.com/auth/shopify/callback`
  - `https://www.fake-shopifytest.com/api/auth/callback`
- Save all changes
- Under the "Overview" tab, note your Client ID and Client secret. You'll need to set these values in local environment variables in the next step.


### Change vite.js Configuration to Support Hot Module Reloading

```
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

### Running the server for app development

- Include localhost info in dev_embed.js:

```
import RefreshRuntime from "http://localhost:3000/@react-refresh";
```

- Change src entry in index.html as follows:

```
<script type="module" src="http://localhost:3000/src/index.jsx"></script>
```

- Install dependencies:

```
npm install
```

- Then run server and frontend in two separate terminals

Server:

```
npm run dev-server
```

where the script is defined as:

```
"dev": "cross-env NODE_ENV=development HOST=https://www.fake-doppio-shopifytest.com NODE_TLS_REJECT_UNAUTHORIZED=0 nodemon index.js",
```


Frontend:

```
npm run dev-frontend
```

where the script is defined as:

```
"dev": "vite --port 3000"
```

After the server is spun up, use this URL to install the app:

```
https://www.fake-shopifytest.com/?shop={your-store}.myshopify.com
```

You can get the shop link from the shopify store page.
Note that you need to uninstall the app on a normal shopify page first if you have previously installed it
before accessing the installation link.
