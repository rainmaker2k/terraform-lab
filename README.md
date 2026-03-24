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
3. Create lambda/lambda_function.py and paste the following:


Python
```python
import json

def handler(event, context):  
    return {  
        'statusCode': 200,  
        'body': json.dumps('Hello, World!')  
    }
```
## ---

**📦 Step 2: Define the Lambda Resource**

Create a file named main.tf. Add the provider configuration and the Lambda logic below. This section handles the zipping of your code and the IAM roles required for execution.

Terraform

```hcl
terraform {  
  required_providers {  
    aws = { source = "hashicorp/aws", version = "~> 6.0" }  
    archive = { source = "hashicorp/archive", version = "2.7.1" }  
  }  
}

provider "aws" {  
  region = "eu-central-1"  
}

# 1. Package the code automatically  
data "archive_file" "lambda_zip" {  
  type        = "zip"  
  source_file = "lambda/lambda_function.py"  
  output_path = "lambda_output/lambda_function.zip"  
}

# 2. IAM Role for Lambda  
resource "aws_iam_role" "lambda_exec" {  
  name = "lambda_exec_role"

  assume_role_policy = jsonencode({  
    Version = "2012-10-17"  
    Statement = [{  
      Action = "sts:AssumeRole"  
      Effect = "Allow"  
      Principal = { Service = "lambda.amazonaws.com" }  
    }]  
  })  
}

resource "aws_iam_role_policy_attachment" "lambda_exec_policy" {  
  role       = aws_iam_role.lambda_exec.name  
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"  
}

# 3. The Lambda Function  
resource "aws_lambda_function" "hello_world" {  
  function_name = "hello_world"  
  runtime       = "python3.9"  
  role          = aws_iam_role.lambda_exec.arn  
  handler       = "lambda_function.handler"

  filename         = data.archive_file.lambda_zip.output_path  
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256  
}
```
## ---

**🌐 Step 3: Define the API Gateway**

Append this to your main.tf. This creates the HTTP endpoint and links it to your Lambda.

Terraform

```hcl
# 1. Create the REST API  
resource "aws_api_gateway_rest_api" "hello_api" {  
  name = "HelloAPI"  
}

# 2. Create the /hello path  
resource "aws_api_gateway_resource" "hello_resource" {  
  rest_api_id = aws_api_gateway_rest_api.hello_api.id  
  parent_id   = aws_api_gateway_rest_api.hello_api.root_resource_id  
  path_part   = "hello"  
}

resource "aws_api_gateway_method" "hello_method" {  
  rest_api_id   = aws_api_gateway_rest_api.hello_api.id  
  resource_id   = aws_api_gateway_resource.hello_resource.id  
  http_method   = "GET"  
  authorization = "NONE"  
}

# 3. Integrate with Lambda  
resource "aws_api_gateway_integration" "hello_integration" {  
  rest_api_id             = aws_api_gateway_rest_api.hello_api.id  
  resource_id             = aws_api_gateway_resource.hello_resource.id  
  http_method             = aws_api_gateway_method.hello_method.http_method  
  integration_http_method = "POST"  
  type                    = "AWS_PROXY"  
  uri                     = aws_lambda_function.hello_world.invoke_arn  
}

# 4. Permission for API Gateway to call Lambda  
resource "aws_lambda_permission" "allow_api_gateway" {  
  statement_id  = "AllowExecutionFromAPIGateway"  
  action        = "lambda:InvokeFunction"  
  function_name = aws_lambda_function.hello_world.function_name  
  principal     = "apigateway.amazonaws.com"  
  source_arn    = "${aws_api_gateway_rest_api.hello_api.execution_arn}/*/*"  
}

# 5. Deployment and Stage  
resource "aws_api_gateway_deployment" "hello_deployment" {  
  depends_on  = [aws_api_gateway_integration.hello_integration]  
  rest_api_id = aws_api_gateway_rest_api.hello_api.id  
}

resource "aws_api_gateway_stage" "hello_stage" {  
  deployment_id = aws_api_gateway_deployment.hello_deployment.id  
  rest_api_id   = aws_api_gateway_rest_api.hello_api.id  
  stage_name    = "v1"  
}

output "api_endpoint" {  
  value = "https://${aws_api_gateway_rest_api.hello_api.id}.execute-api.eu-central-1.amazonaws.com/v1/hello"  
}
```
## ---

**🚀 Step 4: Deployment & Testing**

1. Run terraform init.  
2. Run terraform apply and type yes.  
3. Copy the api_endpoint from the output and paste it into your browser.

## ---

**Bonus**
- In the first exercise you created an S3 bucket. Try to change your Terraform setup to make use of the S3 bucket as a remote backend.

**🛠️ Troubleshooting**

| Issue | Solution |
| :---- | :---- |
| **502 Bad Gateway** | Your Python code is missing the statusCode or body keys in the return dictionary. |
| **403 Forbidden** | The aws_lambda_permission resource is missing or incorrect. |
| **Lambda not updating** | Ensure source_code_hash is present in the aws_lambda_function block. |

---

**Would you like me to add a section on how to view the logs in CloudWatch?**

**Sources**  
1. [https://cloving.ai/tutorials/how-to-generate-serverless-functions-using-gpt](https://cloving.ai/tutorials/how-to-generate-serverless-functions-using-gpt)  
2. [https://github.com/alokagarwal/deploy-lambda-with-terraform](https://github.com/alokagarwal/deploy-lambda-with-terraform)  
3. [https://github.com/alokagarwal/deploy-lambda-with-terraform](https://github.com/alokagarwal/deploy-lambda-with-terraform)