---
title: "Aws Lambda"
date: 2020-08-02T05:35:36+02:00
draft: true
toc: false
images:
tags:
  - untagged
---

## Python lambda function AWS With virtualenv:

```
Go to 
lib/python3.8/site-packages
```

```
then do 
zip -r9 ${OLDPWD}/function.zip .
```

```
then back to the folder to add the function to the zip with 
```

```
zip -g function.zip lambda_function.py
```

```
The update the function code
aws lambda update-function-code --function-name getFollowerCount --zip-file fileb://function.zip
```

https://j3kxcrii3j.execute-api.eu-central-1.amazonaws.com/default/getFollowerCount
