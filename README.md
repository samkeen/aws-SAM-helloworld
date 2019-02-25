# aws-SAM-helloworld

This is a sample template for sam-app - Below is a brief explanation of what we have generated for you:

```bash
.
├── README.md                   <-- This instructions file
├── event.json                  <-- API Gateway Proxy Integration event payload
├── hello_world                 <-- Source code for a lambda function
│   ├── __init__.py
│   ├── app.py                  <-- Lambda function code
│   ├── requirements.txt        <-- Lambda function code
├── template.yaml               <-- SAM Template
└── tests                       <-- Unit tests
    └── unit
        ├── __init__.py
        └── test_handler.py
```

# Requirements

* [AWS CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html) already configured with Administrator permission
* [Python 3 installed](https://www.python.org/downloads/)
* [Docker installed](https://www.docker.com/community-edition)
* Optional but highly recommended; Install a Python virtual env for this project 

# Local development

## TL;DR

### Working with a lambda running locally
```bash

# start local server
sam local start-api

#### START-DEV-LOOP ####
# edit files

# run tests
python -m pytest tests/ -v

# re-build
# pip freeze if needed
sam build
# hit localhost URL

# return to START-DEV-LOOP

```

### Working with a deployed lambda
```bash

# edit files

# run tests
python -m pytest tests/ -v

# re-build
# pip freeze if needed
sam build
# package and deploy
sam package
sam deploy

# hit URL APIGwy URL 
```

## Extended Cut 

**Start local server (Invoking function locally through local API Gateway)**
```bash
 sam local start-api  
```
You'll be able to make requests at the localhost URL output by the `start-api` command.

### Iterate on app code

Work on `hello_world/app.py`, then re-build project.  [AWS Lambda requires a flat folder](https://docs.aws.amazon.com/lambda/latest/dg/lambda-python-how-to-create-deployment-package.html) 
with the application as well as its dependencies in  deployment package. When you make changes to your 
source code or dependency manifest, run the following command to build your project local testing and deployment.
```bash
# pip freeze > hello_world/requirements.txt
sam build
```
By default, this command writes built artifacts to `.aws-sam/build` folder.
If your dependencies contain native modules that need to be compiled specifically for the operating system 
running on AWS Lambda, use this command to build inside a Lambda-like Docker container instead:
```bash
sam build --use-container
```

Make request to the localhost URL and changes to app will be reflected.

### Additional Local work flows

**Invoking function locally using a local sample payload**

```bash
sam local invoke HelloWorldFunction --event event.json
```

### More details on `sam local start-api`
If the previous command ran successfully you should now be able to hit the following local endpoint 
to invoke your function `http://localhost:3000/hello`

**SAM CLI** is used to emulate both Lambda and API Gateway locally and uses our `template.yaml` to 
understand how to bootstrap this environment (runtime, where the source code is, etc.) - The following 
excerpt is what the CLI will read in order to initialize an API and its routes:

```yaml
...
Events:
    HelloWorld:
        Type: Api # More info about API Event Source: https://github.com/awslabs/serverless-application-model/blob/master/versions/2016-10-31.md#api
        Properties:
            Path: /hello
            Method: get
```

### Packaging and deployment Details

AWS Lambda Python runtime requires a flat folder with all dependencies including the application. SAM will
use `CodeUri` property to know where to look up for both application and dependencies:

```yaml
...
    HelloWorldFunction:
        Type: AWS::Serverless::Function
        Properties:
            CodeUri: hello_world/
            ...
```

Firstly, we need a `S3 bucket` where we can upload our Lambda functions packaged as ZIP before we deploy 
anything - If you don't have a S3 bucket to store code artifacts then this is a good time to create one:

```bash
aws s3 mb s3://BUCKET_NAME
```

`package` and `deploy` documented 
[here](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-deploying.html)

Next, run `sam package` (*equivalent to `aws cloudformation package`*) to package our Lambda function to S3:

```bash
export BUCKET=<### Your Bucket Name ###>
export SOURCE_TEMPLATE=template.yaml
export PACKAGED_TEMPLATE=sam-packaged.yaml
sam package \
  --template-file $SOURCE_TEMPLATE \
  --s3-bucket $BUCKET \
  --output-template-file $PACKAGED_TEMPLATE
```

Next, running `sam deploy` (*equivalent aws `cloudformation deploy`*) will create a Cloudformation Stack and deploy your SAM resources.

```bash
export PACKAGED_TEMPLATE=sam-packaged.yaml
export STACK_NAME=sam-app
sam deploy \
    --template-file ./$PACKAGED_TEMPLATE \
    --stack-name $STACK_NAME \
    --capabilities CAPABILITY_IAM
    
Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - sam-app

```

> **See [Serverless Application Model (SAM) HOWTO Guide](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-quick-start.html) for more details in how to get started.**

After deployment is complete you can run the following command to retrieve the API Gateway Endpoint URL:

```bash
aws cloudformation describe-stacks \
    --stack-name sam-app \
    --query 'Stacks[].Outputs[?OutputKey==`HelloWorldApi`]' \
    --output table
``` 

### Tailing Logs

To simplify troubleshooting, SAM CLI has a command called sam logs. sam logs lets you fetch logs generated by your Lambda function from the command line. In addition to printing the logs on the terminal, this command has several nifty features to help you quickly find the bug.

`NOTE`: This command works for all AWS Lambda functions; not just the ones you deploy using SAM.

```bash
sam logs -n HelloWorldFunction --stack-name sam-app --tail

2019/02/24/[$LATEST]9705bd8a22184c4599771ca30e295cbb 2019-02-24T19:02:16.977000 START RequestId: b0da4e00-bac1-47b6-9e45-058c2a8235ee Version: $LATEST
2019/02/24/[$LATEST]9705bd8a22184c4599771ca30e295cbb 2019-02-24T19:02:16.978000 END RequestId: b0da4e00-bac1-47b6-9e45-058c2a8235ee
2019/02/24/[$LATEST]9705bd8a22184c4599771ca30e295cbb 2019-02-24T19:02:16.978000 REPORT RequestId: b0da4e00-bac1-47b6-9e45-058c2a8235ee  Duration: 0.50 ms       Billed Duration: 100 ms     Memory Size: 128 MB     Max Memory Used: 55 MB  
2019/02/24/[$LATEST]9705bd8a22184c4599771ca30e295cbb 2019-02-24T19:02:58.763000 START RequestId: 4b405347-0e07-4cf8-957a-0d160b1f8fc7 Version: $LATEST

```

You can find more information and examples about filtering Lambda function logs in the
[SAM CLI Documentation](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-logging.html).

### Testing


Next, we install test dependencies and we run `pytest` against our `tests` folder to run our initial unit tests:

```bash
pip install pytest pytest-mock
python -m pytest tests/ -v
```

## Cleanup

In order to delete our Serverless Application recently deployed you can use the following AWS CLI Command:

```bash
aws cloudformation delete-stack --stack-name sam-app
```

### Step-through debugging

**[Enable step-through debugging docs for supported runtimes]((https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-using-debugging.html))**


# Appendix

## SAM and AWS CLI commands

All commands used throughout this document

```bash
# Generate event.json via generate-event command
sam local generate-event apigateway aws-proxy > event.json

# Invoke function locally with event.json as an input
sam local invoke HelloWorldFunction --event event.json

# Run API Gateway locally
sam local start-api

# Create S3 bucket
aws s3 mb s3://BUCKET_NAME

# Package Lambda function defined locally and upload to S3 as an artifact
sam package \
    --output-template-file packaged.yaml \
    --s3-bucket REPLACE_THIS_WITH_YOUR_S3_BUCKET_NAME

# Deploy SAM template as a CloudFormation stack
sam deploy \
    --template-file packaged.yaml \
    --stack-name sam-app \
    --capabilities CAPABILITY_IAM

# Describe Output section of CloudFormation stack previously created
aws cloudformation describe-stacks \
    --stack-name sam-app \
    --query 'Stacks[].Outputs[?OutputKey==`HelloWorldApi`]' \
    --output table

# Tail Lambda function Logs using Logical name defined in SAM Template
sam logs -n HelloWorldFunction --stack-name sam-app --tail
```

## CodePipeline

see: https://docs.aws.amazon.com/lambda/latest/dg/build-pipeline.html

### CloudFormation role and add the AWSLambdaExecute policy to that role

```json


```

