# **Lab: Deploying a Serverless API with Terraform**

## **🎯 Lab Objectives**

In this lab, you will use Terraform to provision a serverless architecture on AWS. You will package a Python script, deploy it as an AWS Lambda function, and expose it to the internet using an Amazon API Gateway REST API.

## ---

**🏗️ Architecture Overview**

The setup follows a standard serverless pattern:

1. **API Gateway** acts as the entry point (HTTPS).  
2. **Lambda** contains the business logic (Python).  
3. **IAM** provides the necessary permissions for these services to talk to each other.

## ---

**🛠️ Step 1: Prepare the Lambda Code**

Before running Terraform, we need the application code.

1. Create a directory: mkdir terraform-lambda-lab && cd terraform-lambda-lab  
2. Create a folder for the code: mkdir lambda  
3. Create lambda/lambda\_function.py and paste the following:


Python
```python
import json

def handler(event, context):  
    return {  
        'statusCode': 200,  
        'body': json.dumps('Hello, World\!')  
    }
```
## ---

**📦 Step 2: Define the Lambda Resource**

Create a file named main.tf. Add the provider configuration and the Lambda logic below. This section handles the zipping of your code and the IAM roles required for execution.

Terraform

```hcl
terraform {  
  required\_providers {  
    aws \= { source \= "hashicorp/aws", version \= "\~\> 6.0" }  
    archive \= { source \= "hashicorp/archive", version \= "2.7.1" }  
  }  
}

provider "aws" {  
  region \= "eu-central-1"  
}

\# 1\. Package the code automatically  
data "archive\_file" "lambda\_zip" {  
  type        \= "zip"  
  source\_file \= "lambda/lambda\_function.py"  
  output\_path \= "lambda\_output/lambda\_function.zip"  
}

\# 2\. IAM Role for Lambda  
resource "aws\_iam\_role" "lambda\_exec" {  
  name \= "lambda\_exec\_role"

  assume\_role\_policy \= jsonencode({  
    Version \= "2012-10-17"  
    Statement \= \[{  
      Action \= "sts:AssumeRole"  
      Effect \= "Allow"  
      Principal \= { Service \= "lambda.amazonaws.com" }  
    }\]  
  })  
}

resource "aws\_iam\_role\_policy\_attachment" "lambda\_exec\_policy" {  
  role       \= aws\_iam\_role.lambda\_exec.name  
  policy\_arn \= "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"  
}

\# 3\. The Lambda Function  
resource "aws\_lambda\_function" "hello\_world" {  
  function\_name \= "hello\_world"  
  runtime       \= "python3.9"  
  role          \= aws\_iam\_role.lambda\_exec.arn  
  handler       \= "lambda\_function.handler"

  filename         \= data.archive\_file.lambda\_zip.output\_path  
  source\_code\_hash \= data.archive\_file.lambda\_zip.output\_base64sha256  
}
```
## ---

**🌐 Step 3: Define the API Gateway**

Append this to your main.tf. This creates the HTTP endpoint and links it to your Lambda.

Terraform

```hcl
\# 1\. Create the REST API  
resource "aws\_api\_gateway\_rest\_api" "hello\_api" {  
  name \= "HelloAPI"  
}

\# 2\. Create the /hello path  
resource "aws\_api\_gateway\_resource" "hello\_resource" {  
  rest\_api\_id \= aws\_api\_gateway\_rest\_api.hello\_api.id  
  parent\_id   \= aws\_api\_gateway\_rest\_api.hello\_api.root\_resource\_id  
  path\_part   \= "hello"  
}

resource "aws\_api\_gateway\_method" "hello\_method" {  
  rest\_api\_id   \= aws\_api\_gateway\_rest\_api.hello\_api.id  
  resource\_id   \= aws\_api\_gateway\_resource.hello\_resource.id  
  http\_method   \= "GET"  
  authorization \= "NONE"  
}

\# 3\. Integrate with Lambda  
resource "aws\_api\_gateway\_integration" "hello\_integration" {  
  rest\_api\_id             \= aws\_api\_gateway\_rest\_api.hello\_api.id  
  resource\_id             \= aws\_api\_gateway\_resource.hello\_resource.id  
  http\_method             \= aws\_api\_gateway\_method.hello\_method.http\_method  
  integration\_http\_method \= "POST"  
  type                    \= "AWS\_PROXY"  
  uri                     \= aws\_lambda\_function.hello\_world.invoke\_arn  
}

\# 4\. Permission for API Gateway to call Lambda  
resource "aws\_lambda\_permission" "allow\_api\_gateway" {  
  statement\_id  \= "AllowExecutionFromAPIGateway"  
  action        \= "lambda:InvokeFunction"  
  function\_name \= aws\_lambda\_function.hello\_world.function\_name  
  principal     \= "apigateway.amazonaws.com"  
  source\_arn    \= "${aws\_api\_gateway\_rest\_api.hello\_api.execution\_arn}/\*/\*"  
}

\# 5\. Deployment and Stage  
resource "aws\_api\_gateway\_deployment" "hello\_deployment" {  
  depends\_on  \= \[aws\_api\_gateway\_integration.hello\_integration\]  
  rest\_api\_id \= aws\_api\_gateway\_rest\_api.hello\_api.id  
}

resource "aws\_api\_gateway\_stage" "hello\_stage" {  
  deployment\_id \= aws\_api\_gateway\_deployment.hello\_deployment.id  
  rest\_api\_id   \= aws\_api\_gateway\_rest\_api.hello\_api.id  
  stage\_name    \= "v1"  
}

output "api\_endpoint" {  
  value \= "https://${aws\_api\_gateway\_rest\_api.hello\_api.id}.execute-api.eu-central-1.amazonaws.com/v1/hello"  
}
```
## ---

**🚀 Step 4: Deployment & Testing**

1. Run terraform init.  
2. Run terraform apply and type yes.  
3. Copy the api\_endpoint from the output and paste it into your browser.

## ---

**🛠️ Troubleshooting**

| Issue | Solution |
| :---- | :---- |
| **502 Bad Gateway** | Your Python code is missing the statusCode or body keys in the return dictionary. |
| **403 Forbidden** | The aws\_lambda\_permission resource is missing or incorrect. |
| **Lambda not updating** | Ensure source\_code\_hash is present in the aws\_lambda\_function block. |

---

**Would you like me to add a section on how to view the logs in CloudWatch?**

**Sources**  
1\. [https://cloving.ai/tutorials/how-to-generate-serverless-functions-using-gpt](https://cloving.ai/tutorials/how-to-generate-serverless-functions-using-gpt)  
2\. [https://github.com/alokagarwal/deploy-lambda-with-terraform](https://github.com/alokagarwal/deploy-lambda-with-terraform)  
3\. [https://github.com/alokagarwal/deploy-lambda-with-terraform](https://github.com/alokagarwal/deploy-lambda-with-terraform)