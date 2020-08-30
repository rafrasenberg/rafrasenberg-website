---
title: "AWS Lambda function to retrieve Twitter follower count"
date: 2020-08-29T09:28:55+02:00
draft: false
toc: false
images:
tags:
  - AWS
  - Python
---

## Intro :point_down:
One of my goals for this current year is to get more comfortable with AWS and get certified eventually. Therefore I will be trying out different things and blogging about it here.

I am starting out with a quite simple task and that is deploying an **AWS Lambda function**. In this case an AWS Lambda function that retrieves the count of your current Twitter followers. 

Everytime the AWS API endpoint gets called, a Python script will run that will retrieve the current stats from the Twitter API and then returns it in JSON format so it can be easily used across systems.

**Disclaimer:** I am not an expert in AWS. So I might not use best practices everywhere.

## What is an AWS Lambda function? :large_orange_diamond:

From the docs:

> AWS Lambda is a serverless compute service that runs your code in response to events and automatically manages the underlying compute resources for you.

The code you run on AWS Lambda is called a "Lambda function". Lambda **runs your code on high-availability compute infrastructure** and performs all the administration of the compute resources. So for example server and operating system maintenance, capacity provisioning and automatic scaling, code and security patch deployment, and code monitoring and logging.

When you create and deploy your Lambda function, it is always ready to run as soon as you trigger it. These triggers you set-up yourself. Some examples that you can use to trigger your Lambda function:

- HTTP requests via the Amazon API Gateway
- Modifications to objects in Amazon S3
- Table updates in Amazon DynamoDB
- State transitions in AWS Step Functions

In this post we will use the first one, a HTTP request via the Amazon API Gateway.

## Setting up the base for the function :cool:

In order to set-up the Lambda function you need to create an AWS account. Login using your existing account or sign up for a new account using [this link](https://console.aws.amazon.com/).

After the initial set-up of your account, follow the menu link ***Services***, then under the heading ***Compute*** click on the link ***Lambda***.

On the left side you are greeted with a small menu, go to ***Functions***. Here we will create the base of the function and give it a name. Later on we are deploying our actual Python code from the command line using the AWS CLI tool.

![create AWS Lambda function](/images/blog/1/image1.png)

Name the function `getTwitterFollowerCount` and at runtime choose Python 3.8, since we are using Python code later on. AWS Lambda has all the major languages available, so you can easily pick another one in a later stage if you would prefer that.

At the permission section make sure you pick the first option: "Create a new role with basic Lambda permissions". Now finalize it and create the actual function by going to the top-right side of the page and click ***Create Function***. 

Congrats you have made your first AWS Lambda Function! :trophy:

## Testing our new function :white_check_mark:

Alrighty, time to test our new function. If everything went correct you will be greeted with a configuration panel of the function you have just made, similar to the one below.

![controlpanel AWS Lambda function](/images/blog/1/image2.png)

As you can see in the function code, every function that gets created comes with some boilerplate code that includes a Lambda handler. The Lambda handler is the function in your code, that AWS Lambda can invoke when the service executes your code. For Python the general syntax structure looks like this:

```python
def handler_name(event, context): 
    ...
    return some_value
```

You always need to specify a handler, otherwise your function won't run and the deploy build will fail.

As you can see at the function code in the AWS console, the default Lambda handler that got generated when we created the function looks like this:

```python
def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'body': json.dumps('Hello from Lambda!')
    }
```

Let's see what is going on here. Whenever the Lambda function gets executed, it will run this Python function and then return a `statuscode 200` and a JSON response saying `Hello from Lambda!`

Time to test this!

Click the ***Test*** button at the right top of the function configuration panel. You will see a pop-up like the one below. We can use all the default settings here for this small test. The only thing we need to do is entering `testEvent` at the "Event name" field. Once done, click ***Create*** at the bottom right.

![test event for AWS Lambda function](/images/blog/1/image3.png)

All done. Now all that is left is the actual test of the function. Click the ***Test*** button again and look at the output below. As you can see we ran our function and got our JSON response back, neat! :fire:

There is some additional information that is available to us that you can find back in the logs each time you execute a Lambda function. This is for example: the request ID, the time it took to execute the function, the memory usage etc.

![test log AWS Lambda function](/images/blog/1/image4.png)

So we created the Lambda function, tested it, now it is time to write our actual Python function that contains the logic for retrieving data from the Twitter API regarding our current follower count. 

## Writing our Python code :snake:

Alright the first thing we do is creating a virtual environment for Python, so that all our packages are contained within this envirnoment and not installed globally. Make sure you have installed `virtualenv` and `pip`. There are a lot of resources available on the internet for this, I won't get into that here since it is out of the scope of this blogpost.

So let's create our environment through the command line:

```
$ virtualenv -p python3.8 lambda
```

You might have different Python versions running on your machine, therefore we specify the Python version explicitly when creating the virtual environment, so it matches with our Python version of our Lambda function. After creating the env, activate it and `cd` into it.

```
$ source lambda/bin/activate && cd lambda
```

Now create a `src` folder within the lambda folder, where we will store our Python code and `cd` into it. 

```
$ mkdir src && cd src
```

To retrieve the actual follower count data we will be using a Python package which acts as a Python wrapper around the Twitter API. You can install it using:

```
$ pip install python-twitter
```

Now create a new Python file and start-up VS code (or whatever you prefer):

```
$ code lambda_function.py
```

To access the Twitter API you will need an developer account in order to get API credentials. Head over to [developer.twitter.com](https://developer.twitter.com) and create one. Now use the following code snippet in your Python file:

```python
import twitter

def lambda_handler(event, context):
    CONSUMER_KEY = 'PMRxxgv52052lskdtgsd'
    CONSUMER_SECRET = 'RwiDbasg234lsdSADF235dkAJKIlcGCIcSI'
    ACCESS_TOKEN = '1074466481134323569-ciuXuhDGOIYHfnssaEFCXugPsN'
    ACCESS_TOKEN_SECRET = 'yAIASSldf43efdSXGsj888fghjdffgK68dd'
    SCREEN_NAME = 'rafrasenberg'

    # Create an Api instance.
    api = twitter.Api(consumer_key=CONSUMER_KEY,
                    consumer_secret=CONSUMER_SECRET,
                    access_token_key=ACCESS_TOKEN,
                    access_token_secret=ACCESS_TOKEN_SECRET)

    data = api.GetUser(screen_name=SCREEN_NAME)


    return { 
        'followers' : data.followers_count
    }
```

Make sure to replace the variables with your credentials (I used fake credentials in this post, obviously) and Twitter handle. The Python code is quite simple, let me explain shortly what is going on here:

1. We create an API instance with the credentials
2. We use the `GetUser` API endpoint to get data based on the Twitter handle
3. From the JSON response that we get from Twitter, we get the amount of followers of a user with `data.followers_count`
4. We wrap this API call in a `lambda_handler` and return our own JSON response

So the Python code is working, now back to our Lambda function!

## AWS Lambda deployment package through AWS CLI :repeat:

Make sure you have AWS CLI set-up on your machine to continue with this part. [Check out the docs](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) to see how to install it on different operating systems, I won't be going over that here.

Because we have external dependencies in our Python code (The Twitter API wrapper), we need to turn it into a deployment package. This is required anytime you are using the Lambda API to manage functions, or if you need to include libraries and dependencies other than the AWS SDK. A deployment package is a ZIP archive that contains your function code and dependencies.

So what we want to do first is zip all the site-packages of our Python code. First check your folder structure, on my machine the base folder (the virtual environment with the src files) is located in `~/Development/lambda`. When running the command below, replace all the `~/Development/lambda` parts, with the correct path of your own machine.

```
$ cd ~/Development/lambda/lib/python3.8/site-packages && zip -r9 ~/Development/lambda/src/function.zip . && cd ~/Development/lambda/src
```

If everything went well, you have a freshly generated `function.zip` file now in the `src` folder. :smile: All that is left right now is adding the function code to our archive, by running the command below. 

```
$ zip -g functions.zip lambda_function.py
```

Great! :trophy: Now we have a binary ZIP package that is ready to be deployed to Lambda. What we will do is run an update command. We will use the Lambda function that we created earlier through the AWS console and update that. We use the AWS CLI command `aws lambda update-function-code` for that. We add two arguments, the name of the function, which is `getTwitterFollowerCount` and the location of the ZIP deployment package.

**Note:** If you named your Lambda function differently in the AWS Console earlier, make sure you change this command as well to match the correct name.

```
$ aws lambda update-function-code --function-name getTwitterFollowerCount --zip-file fileb://function.zip
```

After running the above command you will receive a JSON response similar to the one below.

```
{
    "FunctionName": "getTwitterFollowerCount",
    "FunctionArn": "arn:aws:lambda:eu-central-1:001533050737:function:getTwitterFollowerCount",
    "Runtime": "python3.8",
    "Role": "arn:aws:iam::001533050737:role/service-role/getTwitterFollowerCount-role-m8vxr9y6",
    "Handler": "lambda_function.lambda_handler",
    "CodeSize": 4996597,
    "Description": "",
    "Timeout": 3,
    "MemorySize": 128,
    "LastModified": "2020-08-29T11:47:52.701+0000",
    "CodeSha256": "0HPUHFxNWqu6xLl2NistRBoHp6FEBvnBFXnRXjfomXA=",
    "Version": "$LATEST",
    "TracingConfig": {
        "Mode": "PassThrough"
    },
    "RevisionId": "6c16c627-d4e2-4057-b38d-94cb16395f2f",
    "State": "Active",
    "LastUpdateStatus": "Successful"
}
```

## Setting up the API Gateway

The final step we have to take is setting up an API endpoint, so we can actually call our function through a HTTP call. Go back to the AWS console and the function configuration panel. Click the ***Add trigger*** button.

![test log AWS Lambda function](/images/blog/1/image5.png)

In the following screen use the settings below and click the button ***Add***.

![test log AWS Lambda function](/images/blog/1/image6.png)

> ### DONE! :fire::trophy:

All that is left now is checking what our actual HTTP endpoint is and test it, to see if everything works as it should. Check out the API Gateway at the bottom of the dashboard and expand it by pressing ***Details***. As you can see we can find the API endpoint there. 

![test log AWS Lambda function](/images/blog/1/image7.png)

Fire up the command line and use something like `curl` to test for the response. 

```
$ curl 'https://4k1g4gaqz2.execute-api.eu-central-1.amazonaws.com/default/getTwitterFollowerCount'
```

If everything went well, you just received a nice JSON response with your current follower count!

```
$ {
  "followers": 18311
}
```

Easy peasy, lemon squeezy. You now have a serverless AWS Lambda function running that you can call from anywhere you want without the need of any server. Awesome.

See you next time. :wave:
