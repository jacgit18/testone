Sure! Here's a cleaner and better-formatted version of the IAM roles and policies you'll need when testing AWS Step Functions with Lambda.

🛠️ Goal: Set up two IAM roles for testing Step Function ↔ Lambda interactions.

Overview:

| Resource         | Needs Permission To…          | Trusts…                 |
|------------------|-------------------------------|-------------------------|
| Step Function    | Invoke Lambda function        | states.amazonaws.com    |
| Lambda Function  | Start Step Function execution | lambda.amazonaws.com    |

---

🔹 1. IAM Role for Step Function (to invoke Lambda)

Trust Policy (states.amazonaws.com):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "states.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Permissions Policy:

Grant permission to invoke a specific Lambda function:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:<region>:<account-id>:function:<LambdaFunctionName>"
    }
  ]
}
```

During testing, you can temporarily use "Resource": "*" if you're invoking multiple functions.

---

🔹 2. IAM Role for Lambda (to start Step Function execution)

Trust Policy (lambda.amazonaws.com):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Permissions Policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "states:StartExecution"
      ],
      "Resource": "arn:aws:states:<region>:<account-id>:stateMachine:<StepFunctionName>"
    }
  ]
}
```

Optional for debugging:

Add these actions if you want Lambda to check execution results:

- states:DescribeExecution
- states:GetExecutionHistory

---

⚠️ Optional: Dev-Only Full Access Policy

Use this only in a sandbox environment for fast prototyping:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "states:*",
        "lambda:*"
      ],
      "Resource": "*"
    }
  ]
}
```

Attach this to a role called something like DevStepLambdaPlayground.



Great — if you're working with LocalStack to test AWS Step Functions and Lambda locally, here’s how you can structure your setup:

🧩 Goal Recap

- Test Step Function triggering Lambda and vice versa in LocalStack.
- Set up IAM roles with best-practice boundaries.
- Use AWS CLI (pointed at LocalStack) to deploy resources.

Let’s break it down:

---

🔧 STEP 0: Configure AWS CLI for LocalStack

First, configure your local AWS CLI to point to LocalStack:

```bash
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1
```

If using LocalStack Pro with HTTPS, update endpoints accordingly.

---

🔧 STEP 1: Create IAM Roles

Role for Step Function to invoke Lambda:

create-role-step-function.json

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "states.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Create role:

```bash
aws --endpoint-url=http://localhost:4566 iam create-role \
  --role-name StepFunctionExecutionRole \
  --assume-role-policy-document file://create-role-step-function.json
```

Attach policy to allow Lambda invocation:

```bash
aws --endpoint-url=http://localhost:4566 iam put-role-policy \
  --role-name StepFunctionExecutionRole \
  --policy-name StepFunctionInvokeLambdaPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": "lambda:InvokeFunction",
        "Resource": "*"
      }
    ]
  }'
```

---

Role for Lambda to start a Step Function:

create-role-lambda.json

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "lambda.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Create Lambda role:

```bash
aws --endpoint-url=http://localhost:4566 iam create-role \
  --role-name LambdaExecutionRole \
  --assume-role-policy-document file://create-role-lambda.json
```

Attach policy to allow Step Function execution:

```bash
aws --endpoint-url=http://localhost:4566 iam put-role-policy \
  --role-name LambdaExecutionRole \
  --policy-name LambdaStartStepFunctionPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "states:StartExecution"
        ],
        "Resource": "*"
      }
    ]
  }'
```

---

🔧 STEP 2: Deploy Lambda Functions

Assuming you already zipped your code:

```bash
aws --endpoint-url=http://localhost:4566 lambda create-function \
  --function-name HelloLambda \
  --runtime python3.9 \
  --role arn:aws:iam::000000000000:role/LambdaExecutionRole \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://lambda.zip
```

---

🔧 STEP 3: Create a Step Function

sample-sfn-definition.json:

```json
{
  "Comment": "A simple state machine",
  "StartAt": "InvokeLambda",
  "States": {
    "InvokeLambda": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:000000000000:function:HelloLambda",
      "End": true
    }
  }
}
```

Create Step Function:

```bash
aws --endpoint-url=http://localhost:4566 stepfunctions create-state-machine \
  --name HelloStateMachine \
  --role-arn arn:aws:iam::000000000000:role/StepFunctionExecutionRole \
  --definition file://sample-sfn-definition.json
```

---

🚀 STEP 4: Trigger & Test

Invoke Step Function:

```bash
aws --endpoint-url=http://localhost:4566 stepfunctions start-execution \
  --state-machine-arn arn:aws:states:us-east-1:000000000000:stateMachine:HelloStateMachine \
  --name testExecution \
  --input '{"message": "hello"}'
```

From Lambda (if configured in reverse): you can also call startExecution using boto3 in your Lambda code.

---

🧼 Best Practices in LocalStack Context

- Keep policies permissive in local/dev but scoped in prod.
- Use Resource: "*" during testing, but log/print inputs and outputs clearly.
- Keep IAM roles in reusable scripts or infrastructure-as-code (CDK, Terraform).
- Use LocalStack's "awslocal" wrapper (optional):

  Example:

  ```bash
  awslocal lambda invoke ...
  ```

---

Need a Docker Compose template to spin up LocalStack with Step Functions and Lambda enabled? I can drop that too.

