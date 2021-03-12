---
title: "Automating AWS Lambda container image deployments with Terraform"
description: "Learn how to automate AWS Lambda container image deployments with Terraform, using your own custom made module."
date: 2021-02-28T16:54:36+01:00
draft: true
toc: false
images:
  - /images/blog/8/header-image.png
tags:
  - aws
  - serverless
  - terraform
  - docker
---

---

## Introduction :pushpin:

Last week we dove into AWS Lambda container images. This post is building upon that, automating our deployment workflow using Terraform and a custom made module.

If you haven't checked out the previous post yet, and you're new to Lambda & containers, I advise to check that out first. I'll be moving quickly, so following along will be easier if you read the previous one.

You can find that post [here](https://rafrasenberg.com/posts/deploying-fastapi-on-aws-as-a-lambda-container-image/).

> Article experience level: **Advanced**

I categorize every article based on complexity. It's a good way to indicate how well you can follow along with the article since it determines how deep I will explain certain concepts.

**Prerequisites**

- Docker installed
- Terraform installed
- AWS account
- AWS CLI set-up

:point_right: [The Github repo for this blog post](https://github.com/rafrasenberg/lambda-container-image-terraform)

## The problem

As we saw in the previous post, manually deploying Lambda functions as container images was a rather cubersome process and not very efficient. We had to manually build our image first, push to ECR, get the digest URL and then deploy with serverless.

When we see that as an engineer, we are all thinking the same:

**That can be automated!**

And indeed we can.

Ideally you want something that integrates nicely with your existing IaC tools, and not rely too much on things like the serverless framework. That's why we are going to create our own custom "severless" module in Terraform, to automate our deployments.

Let's start! :rocket:

## Setting up our workspace

Let's start with our folder structure. To simplify this process or if you are lazy, you can visit the Github repo and fork/clone it.

Make sure your project has the following folders and files. We'll go over each of them and fill them up throughout this post.

```
.
├── app/
│   ├── src/
│   │   └── main.py
│   └── Dockerfile.prod
└── ops/
    ├── modules/
    │   └── serverless/
    │       ├── main.tf
    │       ├── outputs.tf
    │       ├── variables.tf
    │       └── providers.tf
    ├── backend.tf
    ├── outputs.tf
    └── main.tf
```

What we do here is split up our project into seperatera folders. One `app` folder, which will hold all of our application files that we want to include into our image. And then an `ops` folder, where our Terraform infrastructure resides.

## Custom Terraform module

A Terraform module is a container for multiple resources that are used together. Modules can be used to create lightweight abstractions, so that you can describe your infrastructure in terms of its architecture, rather than directly in terms of physical objects.

The `.tf` files in your working directory, when you run `terraform plan` or `terraform apply` together form the root module. That module may call other modules and connect them together by passing output values from one to input values of another.

This makes structuring your architecture into logical, re-usable pieces very easy and it creates a clean, maintainable code base.

#### Our own Serverless module

So let's start creating our custom "serverless" module. `cd` into the `serverless` module folder and start with `providers.tf`. We are going to use the `aws` provider and a `docker` module.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.30.0"
    }
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 2.11.0"
    }
  }
}
```

## ECR repository

Switch to `main.tf` and the first thing we will do is create an ECR repo that will hold our container image. We use some nifty tricks to store the the current `ecr_address` and `ecr_image` as a local. A local value assigns a name to an expression, so you can use it multiple times within a module without repeating it.

We also use one variable here called `var.application_image_version`, which we will make dynamic so we can deploy different versions of our image build.

**Note:** I saw the usage of `this` for naming Terraform resources that are used only once in another repo and I really like it!

It makes sense when you know you will only make one of the same resource within one module / project. It's a reference to [JavaScripts `this` keyword](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this) and it allows for easy referencing in your files. Feel free to change that to something of your liking, of course.

```hcl
resource "aws_ecr_repository" "this" {
  name = var.ecr_repository_name
}

data "aws_caller_identity" "this" {}
data "aws_region" "current" {}
data "aws_ecr_authorization_token" "token" {}

locals {
  ecr_address = format("%v.dkr.ecr.%v.amazonaws.com", data.aws_caller_identity.this.account_id, data.aws_region.current.name)
  ecr_image   = format("%v/%v:%v", local.ecr_address, aws_ecr_repository.this.id, var.application_image_version)
}
```

#### Docker image build

```hcl
provider "docker" {
  registry_auth {
    address  = local.ecr_address
    username = data.aws_ecr_authorization_token.token.user_name
    password = data.aws_ecr_authorization_token.token.password
  }
}

resource "docker_registry_image" "this" {
  name = local.ecr_image

  build {
    context    = var.application_path
    dockerfile = var.application_dockerfile_name
    no_cache   = var.application_image_no_cache
  }
}
```

#### Development container

Let's start with our Docker files. What we ideally want is a container for development, in which we will run our FastAPI development server. And then a separate container which will be our production image that will be deployed to AWS. :cloud:

For our development container we'll use the Python 3.9 alpine image:

```docker
FROM python:3.9-alpine

WORKDIR /usr/src/app

COPY requirements/dev.txt ./

RUN python3 -m ensurepip
RUN pip install -r dev.txt

ENV PYTHONPATH "${PYTHONPATH}:/usr/src/app/src"

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

It will install our `dev` requirements, set our Python path inside the container for easy imports, and it will run the FastAPI development server on port 8000.

The dev requirements will contain the following:

```
mangum==0.10.0
fastapi==0.63.0
uvicorn==0.13.3

## Add whatever you need in your dev env
# pytest
# coverage
# black
# etc...
```

As you can see we are using Mangum here.

This is an adapter for using ASGI applications with AWS Lambda & API Gateway. It is intended to provide an easy-to-use, configurable wrapper for any ASGI application, deployed in an AWS Lambda function to handle API Gateway requests and responses.

This will allow us to develop our API using FastAPI just as if we would when deploying it on a traditional server. :fire:

You can extend your development container as much as you want, installing all your preferred dev tools and testing frameworks. Since we are locking this into `dev.txt`, our lambda image for deployment will be nice and slim, containing only the stuff we **really** need in production.

#### Deployment container

AWS has certain requirements for the Docker images, so for our deployment container we will use the AWS-Provided base image for Python 3.8

```docker
FROM public.ecr.aws/lambda/python:3.8

COPY requirements/prod.txt ${LAMBDA_TASK_ROOT}

RUN python3 -m ensurepip
RUN pip install -r prod.txt

ADD src ${LAMBDA_TASK_ROOT}

CMD ["main.handler"]
```

Here we copy our files into the `LAMBDA_TASK_ROOT` which is a runtime environment variable provided by AWS and translates to the path of our Lambda function code.

In the end, we attach our FastAPI handler to the image with `main.handler`.

#### Docker compose file

To avoid long and repetitive Docker commands we use docker-compose. Add the following config to your `docker-compose.yml` file:

```yml
version: "3.8"

services:
  lambda-fastapi-prod:
    build:
      context: .
      dockerfile: ./compose/prod/Dockerfile
    image: <specific-image-name>
    container_name: lambda-fastapi-prod
    ports:
      - 9000:8080

  lambda-fastapi-dev:
    build:
      context: .
      dockerfile: ./compose/dev/Dockerfile
    image: lambda-fastapi-dev:latest
    container_name: lambda-fastapi-dev
    volumes:
      - ./src:/usr/src/app/src
    ports:
      - 8000:8000
```

The `image` tag for the production container we will fill in later since this needs to be a specific tag so that we can use it with AWS ECR.

## Create the FastAPI server

Alright on to the fun part! Let's create our FastAPI server. In `src/main.py` put the following code:

```python
from fastapi import FastAPI
from mangum import Mangum

app = FastAPI(
    title="My Awesome FastAPI app",
    description="This is super fancy, with auto docs and everything!",
    version="0.1.0",
)


@app.get("/ping", name="Healthcheck", tags=["Healthcheck"])
async def healthcheck():
    return {"Success": "Pong!"}


handler = Mangum(app)
```

Here we create a simple `/ping` endpoint to test our app with. As you can see, since FastAPI runs on Starlette we can use async Python here! How cool is that. I can see the async king Node.js, crying in the corner already. :smile:

As you can see, Mangum wraps around the FastAPI app and is specified as the handler, to allow usage on Lambda.

## Run the development server

Time to build our container and run our development server! Within the root directory of your project:

```
docker-compose up lambda-fastapi-dev
```

This will build the development container and run it. Here I am specifically not detaching the container so we can use the terminal for debugging purposes since FastAPI will log output there. :muscle:

The reason we are only running and building our dev container here is because we didn't specify the image tag of our deployment container yet, which we will do later on.

With your image build and FastAPI running in your container, visit `localhost:8000/ping` to check if everything is working. If all went well, you are greeted with our health check message `pong`. :tennis:

#### Auto interactive docs

A really neat feature that's built into FastAPI, is the interactive auto doc feature. Visit `localhost:8000/docs` to see it in action.

Try firing the endpoints, you can do it straight from the docs! Awesome right? :smile:

You can focus on writing your API code, while FastAPI is taking care of the documenting part. This becomes especially useful when your app scales and you are working with larger, decoupled teams.

## Using AWS ECR

To deploy our Lambda image, we need our container image to be registered at AWS's Elastic Container Registry. From here on I am assuming you have an AWS account with the proper rights, and you have the AWS CLI set-up. If you haven't yet, do that first to continue!

First, we need to create a new ECR repository:

```
aws ecr create-repository --repository-name lambda-fastapi --region eu-west-1
```

From the output of this command search for `repositoryUri`. Note this. In my case this is:

```json
"repositoryUri": "105477761364.dkr.ecr.eu-west-1.amazonaws.com/lambda-fastapi"
```

What we do next is logging into ECR with Docker. Take the first part of the `repositoryUri` and use that at the end of the command, like below.

For the observant people, yes that's your AWS account number. ECR URLs always follow this same structure: `<account-number>.dkr.ecr.<your-region>.amazonaws.com`.

```
aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin 105477761364.dkr.ecr.eu-west-1.amazonaws.com
```

#### Building our production image

Alright, with that out of the way, let's go back to our `docker-compose.yml` and add the image tag for the production container.

```yml
---
services:
  lambda-fastapi-prod:
    build:
      context: .
      dockerfile: ./compose/prod/Dockerfile
    image: 105477761364.dkr.ecr.eu-west-1.amazonaws.com/lambda-fastapi:latest
    container_name: lambda-fastapi-prod
    ports:
      - 9000:8080
```

As you can see, we use the ECR repository name here, attached with a tag `:latest`. This is just basic Docker tagging and allows for easy version control. You can, of course, change that to whatever suits you.

Now time to build our container:

```
docker-compose up --build
```

This will build both containers and run them. I hear you thinking now. With my container running in the background;

_Would it be possible?_  
_Can I invoke my production container locally?_

Yes, you can! :rocket:

To make that work, we need to mimic an AWS API gateway event.

```
curl -XPOST "http://localhost:9000/2015-03-31/functions/function/invocations" -d '{
    "resource": "/local-testing",
    "path": "/ping",
    "httpMethod": "GET",
    "headers": {
      "Accept": "*/*",
      "Accept-Encoding": "gzip, deflate",
      "cache-control": "no-cache",
      "CloudFront-Forwarded-Proto": "https",
      "CloudFront-Is-Desktop-Viewer": "true",
      "CloudFront-Is-Mobile-Viewer": "false",
      "CloudFront-Is-SmartTV-Viewer": "false",
      "CloudFront-Is-Tablet-Viewer": "false",
      "CloudFront-Viewer-Country": "US",
      "Content-Type": "application/json",
      "headerName": "headerValue",
      "Host": "gy415nuibc.execute-api.us-east-1.amazonaws.com",
      "Postman-Token": "9f583ef0-ed83-4a38-aef3-eb9ce3f7a57f",
      "User-Agent": "PostmanRuntime/2.4.5",
      "Via": "1.1 d98420743a69852491bbdea73f7680bd.cloudfront.net (CloudFront)",
      "X-Amz-Cf-Id": "pn-PWIJc6thYnZm5P0NMgOUglL1DYtl0gdeJky8tqsg8iS_sgsKD1A==",
      "X-Forwarded-For": "54.240.196.186, 54.182.214.83",
      "X-Forwarded-Port": "443",
      "X-Forwarded-Proto": "https"
    },
    "multiValueHeaders":{
      "Accept":[
        "*/*"
      ],
      "Accept-Encoding":[
        "gzip, deflate"
      ],
      "cache-control":[
        "no-cache"
      ],
      "CloudFront-Forwarded-Proto":[
        "https"
      ],
      "CloudFront-Is-Desktop-Viewer":[
        "true"
      ],
      "CloudFront-Is-Mobile-Viewer":[
        "false"
      ],
      "CloudFront-Is-SmartTV-Viewer":[
        "false"
      ],
      "CloudFront-Is-Tablet-Viewer":[
        "false"
      ],
      "CloudFront-Viewer-Country":[
        "US"
      ],
      "":[
        ""
      ],
      "Content-Type":[
        "application/json"
      ],
      "headerName":[
        "headerValue"
      ],
      "Host":[
        "gy415nuibc.execute-api.us-east-1.amazonaws.com"
      ],
      "Postman-Token":[
        "9f583ef0-ed83-4a38-aef3-eb9ce3f7a57f"
      ],
      "User-Agent":[
        "PostmanRuntime/2.4.5"
      ],
      "Via":[
        "1.1 d98420743a69852491bbdea73f7680bd.cloudfront.net (CloudFront)"
      ],
      "X-Amz-Cf-Id":[
        "pn-PWIJc6thYnZm5P0NMgOUglL1DYtl0gdeJky8tqsg8iS_sgsKD1A=="
      ],
      "X-Forwarded-For":[
        "54.240.196.186, 54.182.214.83"
      ],
      "X-Forwarded-Port":[
        "443"
      ],
      "X-Forwarded-Proto":[
        "https"
      ]
    },
    "queryStringParameters": {
    },
    "multiValueQueryStringParameters":{
    },
    "pathParameters": {
    },
    "stageVariables": {
      "stageVariableName": "stageVariableValue"
    },
    "requestContext": {
      "accountId": "12345678912",
      "resourceId": "roq9wj",
      "stage": "testStage",
      "requestId": "deef4878-7910-11e6-8f14-25afc3e9ae33",
      "identity": {
        "cognitoIdentityPoolId": null,
        "accountId": null,
        "cognitoIdentityId": null,
        "caller": null,
        "apiKey": null,
        "sourceIp": "192.168.196.186",
        "cognitoAuthenticationType": null,
        "cognitoAuthenticationProvider": null,
        "userArn": null,
        "userAgent": "PostmanRuntime/2.4.5",
        "user": null
      },
      "resourcePath": "/ping",
      "httpMethod": "GET",
      "apiId": "gy415nuibc"
    },
    "body": "{}",
    "isBase64Encoded": false
}'
```

Output:

```
{"isBase64Encoded": false, "statusCode": 200, "headers": {"content-length": "19", "content-type": "application/json"}, "body": "{\"Success\":\"Pong!\"}"}%
```

That's absolutely insane! :fire: We can invoke our Lambda function locally, just as how it would run on AWS itself.

The only things you need to adjust in the event are the `httpMethod` and the `path` and `resourcePath`. You can test every event locally, even `POST` and `PUT` requests, by adding a `body` to it.

The potential of this is huge, because you can easily incorporate integration tests for your Lambda functions now. After all, it runs against the actual Lambda runtime. No more mocking! :ok_hand:

## Deploying our Lambda container

Next up, is deploying our container image. Let's first start with pushing our locally build image to ECR.

```
docker push 105477761364.dkr.ecr.eu-west-1.amazonaws.com/lambda-fastapi:latest
```

Here you would obviously, use the same url as how you tagged your image and build it. We get our output back:

```
latest: digest: sha256:23ea6742e439757b0d331f08c461b0146bd1a6e0796f7dd388ae86abadf54c19 size: 2413
```

Note the digest output, we will use it in our serverless configuration to deploy our Lambda.

With our FastAPI lambda container deployed on ECR, let's create our `serverless.yml` file and deploy the API gateway that we will use to invoke our Lambda.

```yml
service: lambda-fastapi

frameworkVersion: "2"

provider:
  name: aws
  stage: ${opt:stage}
  region: eu-west-1
  lambdaHashingVersion: 20201221
  memorySize: 256
  timeout: 30
  apiName: ${self:service}-${opt:stage}
  apiGateway:
    description: REST API ${self:service}
    metrics: true

functions: ${file(functions.yml):functions}
```

Now add the function in `functions.yml`. Take a good look at the image URL we use here, it follows the following format: `<your-ecr-repo>@<your-digest>`.

Use the digest output here that you noted earlier after the ECR push.

```yml
functions:
  myAwesomeFastAPIFunction:
    image: 105477761364.dkr.ecr.eu-west-1.amazonaws.com/lambda-fastapi@sha256:23ea6742e439757b0d331f08c461b0146bd1a6e0796f7dd388ae86abadf54c19
    events:
      - http:
          path: ping/
          method: get
          cors: true
```

Time to deploy our Lambda! :rocket:

```
serverless deploy --stage dev
```

Output:

```
Service Information
service: lambda-fastapi
stage: dev
region: eu-west-1
stack: lambda-fastapi-dev
resources: 12
api keys:
  None
endpoints:
  GET - https://119dyojuza.execute-api.eu-west-1.amazonaws.com/dev/ping
functions:
  lambda-fastapi: lambda-fastapi-dev-myAwesomeFastAPIFunction
layers:
  None
```

Visit your endpoint and see the result!

```
{"Success":"Pong!"}
```

## Extending the API

Alright, that's very cool what we just did, but you might want a little more than just a `/ping` endpoint. So let's extend our API and truly see the power of Mangum + FastAPI, handling internal routing within a single Lambda container image. :chart_with_upwards_trend:

Add the following files to your `src` folder:

```
src
  api_v1/
    users/
      users.py
    api.py
  main.py
```

As you can see, we can really structure our app very well, by using FastAPI's nested routing. This makes our app look clean and keeps our code separated. You could make multiple modules for different parts of your API.

In our `api.py` file, add the following code:

```python
from fastapi import APIRouter

from .users import users

router = APIRouter()

router.include_router(users.router, prefix="/users", tags=["Users"])
```

Then within our `users.py` file, add the following routes:

```python
from fastapi import APIRouter

router = APIRouter()


@router.get("/{id}")
async def get_user():
    results = {"Success": "This is one user!"}
    return results


@router.get("/")
async def get_users():
    results = {"Success": "All users!"}
    return results
```

And finally, the last step is adding our new router app to our `main.py` file which is our Lambda handler as you remember.

```python
from fastapi import FastAPI
from mangum import Mangum
from api_v1.api import router as api_router

app = FastAPI(
    title="My Awesome FastAPI app",
    description="This is super fancy, with auto docs and everything!",
    version="0.1.0",
)


@app.get("/ping", name="Healthcheck", tags=["Healthcheck"])
async def healthcheck():
    return {"Success": "Pong!"}

app.include_router(api_router, prefix="/api/v1")

handler = Mangum(app)
```

Assuming your development server is still running (this is the beauty of FastAPI development), visit `localhost:8000/api/v1/users`. It works! Awesome. :fire:

Also, check out `/docs` to see our newly updated interactive docs:

![FastAPI autodocs](/images/blog/8/fastapi-docs.png)

As you can see, our new endpoint is working. Note the prefix we use here `/api/v1`.

It's always good practice to version your APIs. When your app grows, you might plan on releasing a new API with breaking changes for your end-user. By versioning it, you could just start developing in a new folder `api_v2` and create a new prefix `/api/v2` for it. This will give your users time to switch over while slowly deprecating your old API.

#### Rebuild the container

Time to rebuild our container, since our code changed. :construction:

```
docker-compose build --no-cache lambda-fastapi-prod
```

Now push the new version to ECR, note the digest output again so we can reference that in our `serverless` file.

```
docker push 105477061364.dkr.ecr.eu-west-1.amazonaws.com/lambda-fastapi:latest
```

#### Adding new routes

With our new container on ECR, add the endpoints to our `functions.yml` file:

```yml
functions:
  myAwesomeFastAPIFunction:
    image: 105477761364.dkr.ecr.eu-west-1.amazonaws.com/lambda-fastapi@sha256:cc85e1e3b0a862115deee4f151e981c79d5b453fb0645099c6e86a29deeea0e0
    events:
      - http:
          path: ping/
          method: get
          cors: true
      - http:
          path: api/v1/users/
          method: any
          cors: true
      - http:
          path: api/v1/users/{id}
          method: any
          cors: true
```

Make sure you change the digest of your image as well, to the latest one that you just deployed. These kinds of things can of course be automated, making deployment even quicker, but that's not the point of this blog post. You can be creative yourself with that. :stuck_out_tongue_closed_eyes:

Also, something to note, the serverless framework itself does offer a functionality where it will build your Docker container for you, basically not having to paste the new digest for the image or push it to ECR yourself. However, this is quite buggy and I didn't get it to work for me.

This also means that each time you might just want to change an API gateway route, it will rebuild the full container and pushes it to ECR, when that's not needed sometimes. So I prefer to do it this way! Feel free to do whatever works for you of course.

Time to deploy! :rocket:

```
serverless deploy --stage dev
```

Output:

```
Service Information
service: lambda-fastapi
stage: dev
region: eu-west-1
stack: lambda-fastapi-dev
resources: 20
api keys:
  None
endpoints:
  GET - https://119dyojuza.execute-api.eu-west-1.amazonaws.com/dev/ping
  ANY - https://119dyojuza.execute-api.eu-west-1.amazonaws.com/dev/api/v1/users
  ANY - https://119dyojuza.execute-api.eu-west-1.amazonaws.com/dev/api/v1/users/{id}
functions:
  lambda-fastapi: lambda-fastapi-dev-myAwesomeFastAPIFunction
layers:
  None
```

Now visit the `users` endpoint first. As you can see, we get our response back that we specified: `{"Success":"All users!"}`

Now let's try our dynamic endpoint by visiting e.g. `users/1`. Our return will be: `{"Success":"This is one user!"}`.

Awesome! As you can see our FastAPI is working perfectly and all internal routing is properly handled.

## Conclusion :zap:

Alright so in this blog post we deployed a FastAPI lambda function on AWS as a container image.

We registered our image on ECR, added multiple internal routes and we deployed it through serverless. Now it's up to you to, of course, to implement the logic behind these API routes. But now you know _how_ to set it up.

I hope you learned enough by now to create container image Lambda APIs on AWS. In the future, I will definitely post some more about it. Maybe integrating our API with various other AWS services like e.g. DynamoDB.

See you next time! :wave:
