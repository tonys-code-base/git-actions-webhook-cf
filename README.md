# Deploy Github-EventBridge Integration using AWS SAM

This repository contains an AWS SAM project for deploying the [Github-EventBridge partner integration](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-saas-furls.html#furls-connection-github). The solution will allow events to be routed from Github to an EventBridge event bus using a Lambda function URL.

# Preparation Steps

## Webhook Secret

Generate a secure webhook secret.

```bash
$ echo `hexdump -n 32 /dev/urandom -v -e '"" 1/1 "%02X" ""'`
```

sample output:

```bash
F379CE1F3736FE95803EBE669AAB77170712DC7B3D146078BE898EB8A784FA7E
```

## Clone this Repo

Clone this repository:

```bash
cd $HOME
git clone https://github.com/tonys-code-base/git-actions-webhook-cf.git git-actions-webhook-cf
cd git-actions-webhook-cf
```

## Prepare Lambda Zip Package

Create Lambda zip package.

The following example creates Lambda zip package file at location  `$HOME/git-actions-webhook-cf/lambda/dist/lambda_src.zip`

```bash
$ cd $HOME/git-actions-webhook-cf/lambda
$ rm -fr dist \
      && mkdir dist \
      && cp {lambda_src.py,requirements.txt} dist \
      && cd dist \
      && pip3 install -r requirements.txt -t . \
      && zip -r lambda_src.zip .
```

## Create S3 Bucket for Lambda Zip Package

Create an S3 bucket for storing the Lambda zip.

Example assumes an S3 bucket name `s3://my-lambda-function-code`.

```bash
$ aws --region us-east-1 s3 mb s3://my-lambda-function-code
```

## Upload Lambda Zip to S3 Bucket

Upload the Lambda zip to the S3 bucket.

```bash
$ cd $HOME/git-actions-webhook-cf/lambda/dist
$ aws --region us-east-1 s3 cp lambda_src.zip s3://my-lambda-function-code/lambda_src.zip
```

## Update CloudFormation Template

Modify the CloudFormation `$HOME/git-actions-webhook-cf/template.yaml`, replacing the following resource names to suit your need.

```yaml
Parameters:
  ...
  ...
  GithubWebhookSecretName:
    ...
    ...
    Default: "<AWS secretsmanager secret name that will be used to store the Github webhook secret>"

  EventBusName:
    ...
    ...
    Default: "<name of the EventBridge event bus that will receive the Github events that are processed by the Lambda Function URL>"

  LambdaFunctionName:
    ...
    ...
    Default: "<lambda function name>"

    ...
    ...
  CWInvocationsAlarmName:
    ...
    ...
    Default: "<cloudwatch alarm name for lambda invocation spikes>"

  S3BucketForLambdaCode:
    ...
    ...
    Default: "<name of the s3 bucket that contains zip file of the lambda packaged>"

  S3KeyLambdaZip:
    ...
    ...
    Default: "s3 key of the zip file containing lambda source"
```

Update the following in the `$HOME/git-actions-webhook-cf/samconfig.toml`

```yaml
version = 0.1

...
...

[default.deploy.parameters]
stack_name = "<desired cloudformation stack name>"
...
...
region = "<aws region>"
profile = "<name of shared credentials profile>"
...
...
```

# Sample Deployment

## Values used for Sample Deployment:

- Location where Lambda zip file has been uploaded to
```
  s3://my-lambda-function-code/lambda_src.zip
```
- Parameters values used in `template.yaml`

```bash
...
...
  GithubWebhookSecretName:
    Type: String
    Description: Github webhook secret name
    Default: "git-actions-webhook-secret"

  EventBusName:
    Type: String
    Description: EventBridge event bus name
    Default: "git-actions-event-bus"

  LambdaFunctionName:
    Type: String
    Description: Lambda function name
    Default: "git-actions-webhook-lambda-function"

  LambdaInvocationThreshold:
    Type: String
    Description: Innovation Alarm Threshold for number of events in a 5 minute period.
    Default: "2000"

  CWInvocationsAlarmName:
    Type: String
    Description: Alarm name for Inbound webhook Lambda invocation traffic spikes
    Default: "git-actions-webhook-invocations-alarm"

  S3BucketForLambdaCode:
    Type: String
    Description: S3 bucket where lambda zip source is located
    Default: "my-lambda-function-code"

  S3KeyLambdaZip:
    Type: String
    Description: S3 key of the lambda zip package
    Default: "lambda_src.zip"
```

- values used in `samconfig.toml`

```bash
version = 0.1

[default.build.parameters]
cached = true
parallel = true

[default.deploy.parameters]
stack_name = "git-actions-webhook"
capabilities = "CAPABILITY_NAMED_IAM"
confirm_changeset = true
resolve_s3 = true
region = "us-east-1"
profile = "my-aws-shared-creds-profile"

[default.sync.parameters]
watch = true
```

## Build/Deploy

### Run the build.

```bash
$ cd $HOME/git-actions-webhook-cf
$ sam build
```

output:

```bash
Starting Build use cache                                                                                                                                                                                                                                                    
The resource AWS::Lambda::Function 'WebhookFunction' has specified S3 location for Code. It will not be built and SAM CLI does not support invoking it locally.                                                                                                             

Build Succeeded

Built Artifacts  : .aws-sam/build
Built Template   : .aws-sam/build/template.yaml

Commands you can use next
=========================
[*] Validate SAM template: sam validate
[*] Invoke Function: sam local invoke
[*] Test Function in the Cloud: sam sync --stack-name {{stack-name}} --watch
[*] Deploy: sam deploy --guided
```

### Validate

```bash
$ sam validate --lint
```

output:

```bash
..........git-actions-webhook-cf/template.yaml is a valid SAM Template
```

### Deploy

- Run the following to begin the deployment. You will be prompted to the value for the parameter `GithubWebhookSecret` that was generated during the preparation steps.

```bash
$ sam deploy --guided
```

- sample log:

```bash
Configuring SAM deploy
======================

        Looking for config file [samconfig.toml] :  Found
        Reading default arguments  :  Success

        Setting default arguments for 'sam deploy'
        =========================================
        Stack Name [git-actions-webhook]: 
        AWS Region [us-east-1]: 
        Parameter GithubWebhookSecret: 
        Parameter GithubWebhookSecretName [git-actions-webhook-secret]: 
        Parameter EventBusName [git-actions-event-bus]: 
        Parameter LambdaFunctionName [git-actions-webhook-lambda-function]: 
        Parameter LambdaInvocationThreshold [2000]: 
        Parameter CWInvocationsAlarmName [git-actions-webhook-invocations-alarm]: 
        Parameter S3BucketForLambdaCode [my-lambda-function-code]: 
        Parameter S3KeyLambdaZip [lambda_src.zip]: 
        #Shows you resources changes to be deployed and require a 'Y' to initiate deploy
        Confirm changes before deploy [Y/n]: 
        #SAM needs permission to be able to create roles to connect to the resources in your template
        Allow SAM CLI IAM role creation [Y/n]: 
        #Preserves the state of previously provisioned resources when an operation fails
        Disable rollback [y/N]: 
The resource AWS::Lambda::Function 'WebhookFunction' has specified S3 location for Code. It will not be built and SAM CLI does not support invoking it locally.   
        Save arguments to configuration file [Y/n]: n

        Looking for resources needed for deployment:
        Creating the required resources...
        Successfully created!

        Managed S3 bucket: aws-sam-cli-managed-default-samclisourcebucket-llllllllllll
        A different default S3 bucket can be set in samconfig.toml and auto resolution of buckets turned off by setting resolve_s3=False

        Deploying with following values
        ===============================
        Stack name                   : git-actions-webhook
        Region                       : us-east-1
        Confirm changeset            : True
        Disable rollback             : False
        Deployment s3 bucket         : aws-sam-cli-managed-default-xxxxxxxxxxxxxxx-yyyyyyyyyyyyyyyyyy
        Capabilities                 : ["CAPABILITY_NAMED_IAM"]
        Parameter overrides          : {"GithubWebhookSecret": "*****", "GithubWebhookSecretName": "git-actions-webhook-secret", "EventBusName": "git-actions-event-bus", "LambdaFunctionName": "git-actions-webhook-lambda-function", "LambdaInvocationThreshold": "2000", "CWInvocationsAlarmName": "git-actions-webhook-invocations-alarm", "S3BucketForLambdaCode": "my-lambda-function-code", "S3KeyLambdaZip": "lambda_src.zip"}
        Signing Profiles             : {}

Initiating deployment
=====================

        Uploading to git-actions-webhook/XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX.template  5067 / 5067  (100.00%)


Waiting for changeset to be created..

CloudFormation stack changeset
-----------------------------------------------------------------------------------------------------------
Operation               LogicalResourceId                     ResourceType                     Replacement 
-----------------------------------------------------------------------------------------------------------
+ Add                   LambdaInvocationsAlarm                AWS::CloudWatch::Alarm           N/A   
+ Add                   WebhookFunctionRole                   AWS::IAM::Role                   N/A   
+ Add                   WebhookFunctionUrlPublicPermissions   AWS::Lambda::Permission          N/A   
+ Add                   WebhookFunctionUrl                    AWS::Lambda::Url                 N/A   
+ Add                   WebhookFunction                       AWS::Lambda::Function            N/A   
+ Add                   WebhookSecretsManager                 AWS::SecretsManager::Secret      N/A   
-----------------------------------------------------------------------------------------------------------


Changeset created successfully. arn:aws:cloudformation:us-east-1:xxxxxxxxxxxx:changeSet/samcli-deploy8888888888/xxxxxxxxx-nnnn-yyyy-zzzz-ttttttttttttt


Previewing CloudFormation changeset before deployment
======================================================
Deploy this changeset? [y/N]: 
```

- respond `y` to the prompt: `Deploy this changeset? [y/N]` to deploy the stack
- sample output log from a successful deployment

```
  Deploy this changeset? [y/N]: y
  
  **************** - Waiting for stack create/update to complete
  
  CloudFormation events from stack operations (refresh every 5.0 seconds)
  -----------------------------------------------------------------------------------------------------------------------------
  ResourceStatus           ResourceType                    LogicalResourceId                     ResourceStatusReason  
  -----------------------------------------------------------------------------------------------------------------------------
  CREATE_IN_PROGRESS       AWS::CloudFormation::Stack      git-actions-webhook                   User Initiated  
  CREATE_IN_PROGRESS       AWS::SecretsManager::Secret     WebhookSecretsManager                 -   
  CREATE_IN_PROGRESS       AWS::SecretsManager::Secret     WebhookSecretsManager                 Resource creation Initiated   
  CREATE_COMPLETE          AWS::SecretsManager::Secret     WebhookSecretsManager                 -   
  CREATE_IN_PROGRESS       AWS::IAM::Role                  WebhookFunctionRole                   -   
  CREATE_IN_PROGRESS       AWS::IAM::Role                  WebhookFunctionRole                   Resource creation Initiated   
  CREATE_COMPLETE          AWS::IAM::Role                  WebhookFunctionRole                   -   
  CREATE_IN_PROGRESS       AWS::Lambda::Function           WebhookFunction                       -   
  CREATE_IN_PROGRESS       AWS::Lambda::Function           WebhookFunction                       Resource creation Initiated   
  CREATE_COMPLETE          AWS::Lambda::Function           WebhookFunction                       -   
  CREATE_IN_PROGRESS       AWS::Lambda::Url                WebhookFunctionUrl                    -   
  CREATE_IN_PROGRESS       AWS::Lambda::Permission         WebhookFunctionUrlPublicPermissions   -   
  CREATE_IN_PROGRESS       AWS::CloudWatch::Alarm          LambdaInvocationsAlarm                -   
  CREATE_IN_PROGRESS       AWS::Lambda::Permission         WebhookFunctionUrlPublicPermissions   Resource creation Initiated   
  CREATE_COMPLETE          AWS::Lambda::Permission         WebhookFunctionUrlPublicPermissions   -   
  CREATE_IN_PROGRESS       AWS::CloudWatch::Alarm          LambdaInvocationsAlarm                Resource creation Initiated   
  CREATE_IN_PROGRESS       AWS::Lambda::Url                WebhookFunctionUrl                    Resource creation Initiated   
  CREATE_COMPLETE          AWS::CloudWatch::Alarm          LambdaInvocationsAlarm                -   
  CREATE_COMPLETE          AWS::Lambda::Url                WebhookFunctionUrl                    -   
  CREATE_COMPLETE          AWS::CloudFormation::Stack      git-actions-webhook                   -   
  -----------------------------------------------------------------------------------------------------------------------------
  
  CloudFormation outputs from deployed stack
  -----------------------------------------------------------------------------------------------------------------------------
  Outputs  
  -----------------------------------------------------------------------------------------------------------------------------
  Key                 LambdaFunctionARN  
  Description         Lambda function ARN.   
  Value               arn:aws:lambda:us-east-1:xxxxxxxxxxxx:function:git-actions-webhook-lambda-function   
  
  Key                 FunctionUrlEndpoint  
  Description         Webhhook Function URL Endpoint   
  Value               https://***************************.lambda-url.us-east-1.on.aws/   
  
  Key                 LambdaFunctionName   
  Description         -  
  Value               git-actions-webhook-lambda-function  
  -----------------------------------------------------------------------------------------------------------------------------


  Successfully created/updated stack - git-actions-webhook in us-east-1
```

- to view the resources created by the stack:

```
  $ sam list resources --stack-name git-actions-webhook --profile <your aws creds profile>
```

  output:

```
------------------------------------------------------------------------------------------------------------------------------
  Logical ID                           Physical ID                                                                      
  ------------------------------------------------------------------------------------------------------------------------------
  LambdaInvocationsAlarm               git-actions-webhook-invocations-alarm                                            
  WebhookFunction                      git-actions-webhook-lambda-function                                              
  WebhookFunctionRole                  git-actions-webhook-lambda-function-role                                         
  WebhookFunctionUrl                   arn:aws:lambda:us-east-1:xxxxxxxxxxxx:function:git-actions-webhook-lambda-function   
  WebhookFunctionUrlPublicPermissions  git-actions-webhook-WebhookFunctionUrlPublicPermissions-yyyyyyyyyyy              
  WebhookSecretsManager                arn:aws:secretsmanager:us-east-1:xxxxxxxxxxxx:secret:git-actions-webhook-secret-xxxxxx   
  ------------------------------------------------------------------------------------------------------------------------------
```
