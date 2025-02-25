---
tags:
  - cloud
  - CapitalOne
  - bestPractices
  - RTIC
author:
  - gitUserNamePlaceHolder
Comments: Placeholder comment any thing else you want to mention about the document.
Purpose: This documentation discusses
Status: 
Started: 2025-02-25
EditDate: 
Relates: 
Peer Reviewed: 0
dg-publish:
---
This document provides a step-by-step guide to setting up **AWS Lambda, IAM (Identity and Access Management), and Step Functions** using **LocalStack**, which is a local AWS cloud emulator. The setup is meant for running and testing AWS services locally without needing an actual AWS account.

When a agent or who ever initiate contract offer from Empath 

goes rules lab coming back with true or false 

Can only process one contract at a time which comes from a frontend through the exchange through the public API invoker getting things like contract id, contract info, and account id along with contract draft which gets sent to rules lab which determines contract eligibility if eligible we update the UCP with the status to enroll and set the payments and publish to one stream for the eventual workflow and do a audit and send a response back to the client this is the immediate workflow. 

When comes to Payments still being decided were its being sent to.  

We just schedule when hooks fire

There is a list of actions associated with each contract which also has one offset we need to separate  those actions by the offset meaning immediate actions fired on the same day and eventual actions which are triggered by hooks based on offset date.

Data lambda => JSON => Enrollment Async Step Function

an offset is 


#todo/CapitalOne
- [ ] Need to mock payload based on schema provided from other team below is rough draft of how it should look may need to set hooks in the future for fulfillment lambda

```json
{
  "List": [
    {
      "name": "rateChange",
      "offset": 16
      "otherData": ...
    },
    {
      "name": "action1",
      "offset": 3 // wait 3 days to execute action 
    },
    {
      "name": "action2", // imediate execute action
      "offset": 0
    },
    {
      "name": "action100",
      "offset": 1
    }
  ]
}
```


## 1. Create a Lambda Function  
- Packages a Python script (`lambda-function.py`) into a ZIP file.
- Uses the AWS CLI (configured to connect to LocalStack) to create a Lambda function named `VerifyCustomer`.
- The function is assigned a role (`lambda-role`), which is required for execution permissions.

```sh
zip -r function.zip lambda-function.py

aws --endpoint-url=http://localhost:4566 lambda create-function \
    --function-name VerifyCustomer \
    --runtime python3.12 \
    --handler lambda_function.lambda_handler \
    --zip-file fileb://function.zip \
    --role arn:aws:iam::000000000000:role/lambda-role
```

## 2. Create IAM Role
- Creates an IAM role (`lambda-role`) with a trust policy (`trust-policy.json`) to allow Lambda execution.

```sh
aws --endpoint-url=http://localhost:4566 iam create-role \
    --role-name lambda-role \
    --assume-role-policy-document file://trust-policy.json
```

## 3. Attach Policy to IAM Role
- Attaches a policy (`policy.json`) to the IAM role.
- Ensures that Lambda has the necessary permissions.
```sh
aws --endpoint-url=http://localhost:4566 iam put-role-policy \
    --role-name lambda-role \
    --policy-name lambda-policy \
    --policy-document file://policy.json
```

```sh
aws --endpoint-url=http://localhost:4566 iam attach-role-policy \
    --role-name lambda-role \
    --policy-arn arn:aws:iam::000000000000:policy/lambda-policy
```

## 4. Verify Policy Attached to IAM Role
- Lists attached policies for the `lambda-role` to confirm that permissions are set correctly.

```sh
aws --endpoint-url=http://localhost:4566 iam list-attached-role-policies \
    --role-name lambda-role
```

## 5. Create a DynamoDB Table
- Sets up a local DynamoDB table (`Offers`) with `OfferId` as the primary key.
```sh
aws --endpoint-url=http://localhost:4566 dynamodb create-table \
    --table-name Offers \
    --attribute-definitions AttributeName=OfferId,AttributeType=S \
    --key-schema AttributeName=OfferId,KeyType=HASH \
    --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5
```

## 6. Create and Update Data Lambda Function
- Packages and updates a second Lambda function (`DataLambda`) responsible for processing data.
```sh
cd /Users/cal141/rtjg/LocalStack/lambdas/DataLambda
zip -r DataLambda.zip data_lambda.py

aws --endpoint-url=http://localhost:4566 lambda update-function-code \
    --function-name DataLambda \
    --zip-file fileb://DataLambda.zip
```

## 7. Update Step Functions State Machine
- Updates an AWS Step Functions state machine (`EnrollmentStateMachine`) using a definition file (`collections-process-offers-enrollment.json`).

```sh
aws --endpoint-url=http://localhost:4566 stepfunctions update-state-machine \
    --state-machine-arn arn:aws:states:us-east-1:000000000000:stateMachine:EnrollmentStateMachine \
    --definition file://collections-process-offers-enrollment.json
```

## 8. Test Step Function Execution
- Starts an execution of the `EnrollmentStateMachine` with an input file (`input.json`). 

```sh
aws --endpoint-url=http://localhost:4566 stepfunctions start-execution \
    --state-machine-arn arn:aws:states:us-east-1:000000000000:stateMachine:EnrollmentStateMachine \
    --input file://input.json
```

### **Purpose of This Setup**

This setup is used to locally develop and test an AWS-based workflow involving:

- Lambda functions (for processing customer data).
- IAM roles and policies (for managing permissions).
- DynamoDB (as a database for storing offers).
- Step Functions (for orchestrating multi-step processes).

By using **LocalStack**, developers can simulate AWS services without incurring costs or requiring internet access.