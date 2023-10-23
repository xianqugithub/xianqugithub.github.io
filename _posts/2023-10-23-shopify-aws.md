---
layout: post
title: Deploy Shopify App using AWS at Scale
tags: [shopify, aws]
comments: true
---

{: .box-success}
While theoretically Shopify apps can be hosted on any platforms, the official Shopify documentation does not provide any guidance on how to deploy to Amazon Web Service(AWS), the most widely used cloud service provider, presumably due to its higher technical complexity compared with using providers such as Fly.IO/Heroku. However, for engineers, having the ability to pick up tools at their own choice gives the unique advantage of optimizing the operations of the app. For example, engineers/dev ops can monitor CPU and memory usages of an ECS task to decide if they need to change the instance number to avoid over spending on infrastructure while getting requests served. Here, I am giving a high-level overview of the AWS tools that can be used for deployment and one example of docker file that can be used.

## Architecture
Even though cloud service providers have already significantly reduced the effort to create and maintain infrastructures, there are still a few key points to keep in mind when considering architectural scalability:
1. **Computation Scalability**: Computational resources need to be able to maintain a high service availability at scale.
2. **Storage Scalability**: There should be a plenty amount of storage space to deal with all application data.
3. **Security**: Security threats can counter all the efforts that devoted in previous resources. One dominant example is Denial of Service (DDOS) attack where a malicious attacker can potentially occupy all the resources. Simply put, no one wants to have a service that can handle a high traffic but is vulnerable at the same time. 

To tackle these issues, we can utilize the following AWS tools:
1. **Computation Scalability**:
   - ***Application Load Balancer(ALB)***: The reason that ALB but not NLB(Network Load Balancer) is used is because ALB makes routing decisions at the application layer (HTTP/HTTPS) but NLB routes at TCP/SSL level. It's therefore easier to configure the whole load balancing using just ALB with Route 53. 
   - ***Elastic Container Service(ECS)***: Compared with EC2, ECS has lower operations burden. Plus, the majority of Shopify app should be generated with Shopify CLI which already contains a docker build file and using a Docker based environment for production is thus recommended. 
2. **Storage Scalability**:
   - ***DynamoDb***: While technically, a local database would work (for example sqlite in the docker image), there is risk of ECS service stop due to high disk usage. It's hard to argue against DynamoDb, a service that has nearly unlimited throughput and storage, to store app session data which is non-relational anyway.
3. **Security**:
   - ***Virtual Private Cloud(VPC)***: Put AWS resources in private virtual network can significantly increase the security of the resources with self-defined security groups. 
   - ***Web Application FireWall(WAF)***: WAF can help protect deployed Shopify app, which is a web application from common web exploits.
   - ***Secrets Manager***: as every Shopify app requires a pair API key and API secret to spin up the application, the credentials need to be stored in a secure place. Hard code these credentials in code base exposes the application with high security risk and by using secrets manager, the credentials can be stored safely and retrieved dynamically during the application building process.

Put everything in the same picture, we can have such architecture:
![ShopifyAWSArchitecture](https://xianqugithub.github.io/assets/img/shopify-aws-architecture.jpeg){: .mx-auto.d-block :}

## Docker File for Container
One of the tricky step during the deployment process is to build the app container properly as the container requires a non-trivial amounts of detailed configurations to work properly. The following code excerpts is just one example to show how everything could possibly be pieced together when AWS architecture is involved:

    FROM public.ecr.aws/sam/build-nodejs16.x:1.76.0-20230303022354

    ENV SCOPES="" # Fill in the scope required by the app
    ENV BACKEND_PORT=8080
    ENV SHOPIFY_CLI_TTY=0 # Optional to make the cli not interactive
    EXPOSE 8080

    # Docker starts to build 
    WORKDIR /app
    COPY . .
    RUN chmod +x /app/build-and-serve-app.sh
    CMD ["sh", "/app/build-and-serve-app.sh"]
    # Docker build finishes

A typical `build-and-serve-app.sh` is as follows. It's easier to use a script when non-trivial manipulations of variables are involved(in this case, retrieve Shopify credentials from Secrets Manager and inject them into environment variable dynamically):

    # Retrive credentials from Secrets Manager for app build 
    sc=$(aws secretsmanager get-secret-value \
             --secret-id ShopifySecret \
             --region us-east-1 | jq -r '.SecretString|fromjson')
        
    export SHOPIFY_API_KEY=$(jq -r '.SHOPIFY_API_KEY' <<< "$sc")
    export SHOPIFY_API_SECRET=$(jq -r '.SHOPIFY_API_SECRET' <<< "$sc")
    
    # Install dependencies
    npm run install
    
    # Build the back-end, depends on the setup, this can even be an no-op when express is used
    npm run build-backend

    # Build the fron-end, this could be something like vite build
    npm run build-frontend
    
    # Serve the app
    npm run serve
        
