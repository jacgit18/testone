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




To create an SNS (Simple Notification Service) in LocalStack and set up the necessary IAM permissions, you can follow these steps:

### 1. **Install LocalStack and dependencies**
   First, make sure you have LocalStack installed on your machine. If you haven't installed it yet, you can use `pip` for Python:

   ```bash
   pip install localstack
   ```

   You also need to have `aws-cli` installed to interact with LocalStack.

   ```bash
   pip install awscli
   ```

   Additionally, you can use `docker` to run LocalStack as a container if needed.

### 2. **Start LocalStack**
   Run LocalStack in your terminal:

   ```bash
   localstack start
   ```

   This will launch LocalStack, and you'll be able to use AWS services locally.

### 3. **Create an SNS Topic**
   Once LocalStack is running, you can create an SNS topic using the `aws` CLI tool. Point to LocalStack using the `--endpoint-url` flag:

   ```bash
   aws --endpoint-url=http://localhost:4566 sns create-topic --name MyTopic
   ```

   This will create an SNS topic named `MyTopic`.

### 4. **Create an IAM Role with Permissions**
   To publish or subscribe to SNS, you need to create IAM policies and roles. In LocalStack, IAM roles are also simulated, but they are essential to handle the permissions properly.

   **Create an IAM Policy that allows SNS actions:**

   Create a JSON file (`sns-policy.json`) for the policy definition:

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "sns:Publish",
           "sns:Subscribe",
           "sns:ListSubscriptions",
           "sns:ListSubscriptionsByTopic"
         ],
         "Resource": "*"
       }
     ]
   }
   ```

   **Create the policy in LocalStack:**

   ```bash
   aws --endpoint-url=http://localhost:4566 iam create-policy --policy-name SNSTopicPolicy --policy-document file://sns-policy.json
   ```

   **Create an IAM Role and Attach the Policy:**

   If you want an IAM role to allow the policy above, you can use the following command to create a role and attach the policy.

   ```bash
   aws --endpoint-url=http://localhost:4566 iam create-role --role-name SNSRole --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"sns.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
   ```

   Then, attach the policy to the role:

   ```bash
   aws --endpoint-url=http://localhost:4566 iam attach-role-policy --role-name SNSRole --policy-arn arn:aws:iam::000000000000:policy/SNSTopicPolicy
   ```

### 5. **Publish to SNS Topic**
   To publish a message to the SNS topic using the created IAM role, you'll need to assume the role or use AWS credentials that have permission to interact with the SNS service. If you're testing locally and just need to send a message to the topic:

   ```bash
   aws --endpoint-url=http://localhost:4566 sns publish --topic-arn arn:aws:sns:us-east-1:000000000000:MyTopic --message "Hello from LocalStack!"
   ```

### 6. **Subscribe to SNS Topic**
   You can subscribe to the SNS topic by providing a protocol (e.g., `email`, `sms`, `http`, `lambda`, etc.). For example, to subscribe an email endpoint:

   ```bash
   aws --endpoint-url=http://localhost:4566 sns subscribe --topic-arn arn:aws:sns:us-east-1:000000000000:MyTopic --protocol email --notification-endpoint test@example.com
   ```

### Summary of IAM Permissions:
- **SNS Actions**: You need permissions for actions like `sns:Publish`, `sns:Subscribe`, etc.
- **IAM Role for SNS**: The IAM role should have permissions to assume actions on SNS resources.
- **Policy Attachment**: You must attach your policy to the appropriate role or user.

LocalStack will simulate these AWS services for testing, but it's important to replicate the IAM policies and roles you would use in production.

Let me know if you'd like help with specific parts of this!





To integrate SNS into an AWS Step Function with PagerDuty notifications, here's a practical overview of how you'd wire that up — including how you'd mock or simulate this locally with LocalStack and what IAM permissions you'd need.

🧩 Use Case Overview  
Let’s say you have a Step Function that orchestrates some workflow — maybe monitoring service health or deployment. When something fails or hits a threshold, you want to notify your team via PagerDuty. SNS acts as the middleman to trigger a PagerDuty alert (via an HTTPS subscription or AWS Lambda).

You can structure it like this:

1. Step Function invokes SNS
2. SNS has a subscription that triggers either:
   - A PagerDuty Events API (via HTTPS endpoint), or
   - A Lambda function that calls PagerDuty with a payload

---

🛠️ Setup Components

1. Step Function state machine
2. SNS topic
3. SNS subscription to PagerDuty (either HTTPS or Lambda)
4. IAM roles for execution
5. (Optional) Test locally with LocalStack + mocked endpoints

---

🔧 1. Create SNS Topic

You can do this in AWS CLI or LocalStack:

```bash
aws --endpoint-url=http://localhost:4566 sns create-topic --name AlertTopic
```

---

🔧 2. Subscribe SNS to PagerDuty

📌 Option 1: HTTPS subscription (directly to PagerDuty Events v2 API)

This is harder to mock locally, but in prod you’d do:

```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:000000000000:AlertTopic \
  --protocol https \
  --notification-endpoint https://events.pagerduty.com/v2/enqueue
```

Note: This requires custom message formatting, which SNS can’t do natively — so this setup is often used with Lambda in between.

📌 Option 2: Lambda → PagerDuty

Create a Lambda function that POSTs to PagerDuty with a message like:

```json
{
  "routing_key": "<YOUR_ROUTING_KEY>",
  "event_action": "trigger",
  "payload": {
    "summary": "Step Function failed",
    "severity": "error",
    "source": "your-system-name"
  }
}
```

Then subscribe this Lambda to the SNS topic:

```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:000000000000:AlertTopic \
  --protocol lambda \
  --notification-endpoint arn:aws:lambda:us-east-1:000000000000:function:PagerDutyNotifier
```

---

⚙️ 3. Step Function Integration

In your Step Function definition (ASL - Amazon States Language):

```json
{
  "Comment": "Trigger SNS alert",
  "StartAt": "Notify",
  "States": {
    "Notify": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:000000000000:AlertTopic",
        "Message": "Something bad happened in the workflow"
      },
      "End": true
    }
  }
}
```

If using LocalStack: Make sure to include the right endpoint override and region in your CLI or SDK.

---

🔐 4. IAM Permissions (for Step Function execution role)

The Step Function needs permissions to publish to SNS:

Here’s an example IAM policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sns:Publish"
      ],
      "Resource": "arn:aws:sns:us-east-1:000000000000:AlertTopic"
    }
  ]
}
```

If using Lambda between SNS and PagerDuty, that Lambda function also needs permission to write to CloudWatch Logs and access secrets if needed.

---

🧪 Local Testing with LocalStack

You can test this locally by:

- Creating SNS topics and subscriptions in LocalStack
- Using a dummy HTTPS endpoint (e.g., webhook.site or a local Express server)
- Replacing PagerDuty with logging while testing locally

---

If you want, I can help generate a full working example (including mock Lambda and Step Function definition) that you can use with LocalStack or deploy to AWS. Just say the word.
