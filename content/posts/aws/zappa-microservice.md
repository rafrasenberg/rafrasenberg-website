---
title: "Deploying a serverless microservice with Flask & Zappa"
description: "In this blog we will take a look at Zappa and deploy a serverless microservice using Flask on AWS in under 3 minutes."
date: 2020-09-12T10:15:46+02:00
draft: false
toc: false
images:
  - /images/blog/2/header.png
tags:
  - AWS
  - Python
  - Flask
  - Zappa
---

## Intro :point_down:
In this blog post we are going to take a look at an awesome deployment tool called Zappa. We will deploy a **serverless microservice** using Flask on AWS in under 3 minutes. Exciting.

Our little microservice will be an API endpoint that returns a dataset of a random person in the form of a JSON response. You can view the [source code here](https://github.com/rafrasenberg/flask-zappa-serverless). Let's start!


#### Prerequisites:
- You need an AWS account
- AWS CLI set-up on your development machine
- Virtualenv installed

## What is Zappa? :mag:
Zappa is a tool that makes it very easy to build and deploy server-less, event-driven Python applications on AWS Lambda + API Gateway. 

Think of it as "serverless" web hosting for your Python apps. That means infinite scaling, zero downtime, zero maintenance - and at a fraction of the cost of your current deployments.

## Setting up the project :white_check_mark:
The first thing we need to do is creating a virtual environment where we install Flask and Zappa. To keep things organised I like to add my `src` folder that contains the app files inside the virtualenv folder.

```python
$ virtualenv -p python3 persongenerator
$ cd persongenerator && source bin/activate
$ mkdir src && cd src
$ pip install flask zappa
```

After you have ran the commands above, create a new file called `app.py` and add the code snippet down below. The Flask code is fairly straight-forward, we create a Flask object and attach a route decorator function to it which defines our path. Which is in this case "/" and thus the homepage. 

```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def index():
    return "Hello, world!", 200

# This is for local development
if __name__ == '__main__':
    app.run()
```

You can confirm that your hello world app is working by running `flask run` and visiting your browser at `localhost:5000`

Alright neat, it's working. Time to deploy!

## Ready, set.. DEPLOY! :rocket:
First make sure that you have created an AWS S3 bucket for this app. You can do so by going to the AWS console and then in the top menu following the link ***Services***. Then under the heading ***Storage*** click on the link ***S3***. 

Click the ***Create bucket*** button and add a bucket. 

**Note:** This bucket will NOT contain website files, as this is not a static website. So the bucket does not need to be public. I will explain why we need this bucket later in the post.

Create a new file in your project root called `zappa_settings.json`. This is basically what will hold our Zappa configuration.

```
{
    "dev": {
        "s3_bucket": "your_s3_bucket",
        "app_function": "app.app"
    }
}
```

This defines an environment called 'dev' (later, you may want to add 'staging' and 'production' environments as well). It also defines the name of the S3 bucket we will be deploying to. And at last it points Zappa to a WSGI-compatible function, in this case, our Flask app object.

And now we are ready to deploy! **Wait a minute.. WHAAAATTTT?!** :scream:

Yup, not joking. Run the following command:

```
$ zappa deploy dev
```

And our microservice is alive. You will see in your command line prompt which URL endpoint. That is quick! :fire::fire:

Alright.. slow down there. What is actually going on behind the scenes when we run the command `zappa deploy dev`?

#### Zappa will:

1. Automatically package up your application and local virtual environment into a Lambda-compatible archive and replace any dependencies with versions precompiled for Lambda
2. Then it will set up the function handler and necessary WSGI Middleware
3. Upload the archive to S3
4. Create and manage the necessary Amazon IAM policies and roles and register it as a new Lambda function
5. Create a new API Gateway resource, create WSGI-compatible routes for it and link it to the new Lambda function
6. And at last, delete the archive from your S3 bucket. Handy!

Now you know why we need that S3 bucket. It is only used as a temporary tool for the deployment as there are no files in there. 

Let's expand our "Hello World" microservice and create a random person generator in the next part.

## Creating a random person generator :busts_in_silhouette:
First, create a file in your project root called `functions.py`. Here we will add the logic for our app, instead of stuffing it into the Flask view.

I fired up Google and searched for a list of popular first and last names. These I added in two text files called `first_name.txt` and `last_name.txt`. You can find these text files in the [Github repo](https://github.com/rafrasenberg/flask-zappa-serverless)! 

So we need a little function that runs and selects one random name out of one of those text files (since we want a random first name & last name). Let's write the code.

```python
import random

def random_line(filename):
    with open(filename) as f:
        lines = f.readlines()
        return random.choice(lines).strip()
```

The function takes the filename as an argument so that we can re-use it for both the first name and last name text file. So we open the file, pick a random line and return it. I called `.strip()` on it as well to make sure there are no blank spaces after the name.

Alright cool, so we have a way to get a random first and last name. Now a random address to complete our dataset! I found a JSON file online which contained 300 random addresses. It was in the following format:

```
{
    "addresses": [
        {
            "address1": "1745 T Street Southeast",
            "address2": "",
            "city": "Washington",
            "state": "DC",
            "postalCode": "20020",
            "coordinates": {
                "lat": 38.867033,
                "lng": -76.979235
            }
        },
        {
            "address1": "6007 Applegate Lane",
            "address2": "",
            "city": "Louisville",
            "state": "KY",
            "postalCode": "40219",
            "coordinates": {
                "lat": 38.1343013,
                "lng": -85.6498512
            }
        }
        ......
    ]
}
```

So we need to write another function that takes a random address from this array and then afterwards we need a final function that joins these two together.

This code is quite easy so we could technically also write all of this in one function or even directly into the Flask view. However I like to seperate my code into small bits which makes everything re-usable and way more error prone. I stick to that principle no matter the size of the app.

Alright, so we need to open the JSON file in our Python function and then return one random address. We can use the following code for that:

```python
import random
import json

def random_address(filename):
    with open(filename) as f:
        data = json.load(f)
        random_address = random.choice(data['addresses'])
        return random_address
```

Looks fairly similar to our previous function right? So we open the JSON file and then call `json.load` which will turn our JSON into a Python dictionary. In order to call `random.choice` we need to have a list. So we use `data['addresses']` to get our desired format and finally, we return our random address!

Let's continue with the last function, which will join these two together into one random dataset.

```python
import random
import json

def get_random_person():
    data = random_address('address.json')
    data['firstName'] = random_line('first_name.txt')
    data['lastName'] = random_line('last_name.txt')
    return data
```

Here we first get a random address and save this into the variable `data`. This variable contains our dictionary. We then use a new key as a subscript and assign its value with our `random_line()` function and thus a random name. Then at last, we return the data. 

The full `functions.py` file will look like this:

```python
import random
import json

def random_line(filename):
    with open(filename) as f:
        lines = f.readlines()
        return random.choice(lines).strip()

def random_address():
    with open('address.json') as f:
        data = json.load(f)
        random_address = random.choice(data['addresses'])
        return random_address

def get_random_person():
    data = random_address() 
    data['firstName'] = random_line('first_name.txt')
    data['lastName'] = random_line('last_name.txt')
    return data
```

Now all that is left is updating our `app.py` file. So here we import the function we made and add an extra Flask endpoint to our microservice. We use the built-in `jsonify` from Flask to turn our Python dictionary into valid JSON. 

```python
from flask import Flask
from flask import jsonify
from functions import get_full_person
app = Flask(__name__)

@app.route('/')
def index():
    return "Hello, world!", 200

@app.route('/random-person')
def get_person():
    data = get_full_person()
    return jsonify(data)

# This is for local development
if __name__ == '__main__':
    app.run()
```

Now all that is left is updating our serverless Lambda function with the Zappa update command down below. It will automatically update all the code for us and your new random person endpoint will be available at *dev/random-person*.

```
$ zappa update dev
```

Let's test it quick with `curl`:

```
$ curl 'https://xdsk3y7413.execute-api.eu-central-1.amazonaws.com/dev/random-person'
```
The response:
```
{
   "address1":"8502 Madrone Avenue",
   "address2":"",
   "city":"Louisville",
   "coordinates":{
      "lat":38.1286407,
      "lng":-85.8678042
   },
   "firstName":"Jennifer",
   "lastName":"Jenkins",
   "postalCode":"40258",
   "state":"KY"
}
```

Working like a charm! :smile:

## Conclusion :zap:
Alright, now you see how easy it is to deploy serverless microservices with Flask and Zappa! 

In my previous blog post we also explored Lambda functions but a deployment tool like Zappa makes it 100x times easier. Next up is adding a custom domain to our microservice, which will be covered in a different post.

See you next time! :wave:
