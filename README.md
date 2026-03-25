Here is the complete, single-block Markdown document. It is structured to guide students through the initial build and the subsequent refactoring into a reusable module.

Markdown

# Lab: Building a Serverless API with Terraform

## 🎯 Lab Objectives  
In this lab, you will use Terraform to provision a serverless architecture on AWS. You will package a Python script, deploy it as an AWS Lambda function, and expose it to the internet using an Amazon API Gateway REST API. Finally, you will refactor this code into a reusable module.

---

## 🏗️ Architecture Overview  
The setup follows a standard serverless pattern:  
1. ****API Gateway**** acts as the entry point (HTTPS).  
2. ****Lambda**** contains the business logic (Python).  
3. ****IAM**** provides the necessary permissions for these services to talk to each other.

---

## 🛠️ Step 1: Prepare the Project Structure  
Before running Terraform, we need the application code that the Lambda will execute.

1. Create a project directory: `mkdir terraform-lambda-lab && cd terraform-lambda-lab`  
2. Create a folder for the code: `mkdir lambda`  
3. Create the file `lambda/lambda_function.py` and paste the following:

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

Create a file named main.tf. Add the provider configuration and the Lambda logic below. This section handles the automatic zipping of your code and the IAM roles required for execution.

Terraform

```hcl
terraform {  
  required_providers {  
    aws = {  
      source  = "hashicorp/aws"  
      version = "~> 6.0"  
    }  
    archive = {  
      source  = "hashicorp/archive"  
      version = "2.7.1"  
    }  
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

Append the following code to your main.tf. This creates the REST API, defines the /hello resource, and sets up the "handshake" between the Gateway and the Lambda.

Terraform

```hcl
# 1. Define the REST API Container  
resource "aws_api_gateway_rest_api" "hello_api" {  
  name = "HelloAPI"  
}

# 2. Create the /hello path  
resource "aws_api_gateway_resource" "hello_resource" {  
  rest_api_id = aws_api_gateway_rest_api.hello_api.id  
  parent_id   = aws_api_gateway_rest_api.hello_api.root_resource_id  
  path_part   = "hello"  
}

# 3. Define the HTTP Method (GET)  
resource "aws_api_gateway_method" "hello_method" {  
  rest_api_id   = aws_api_gateway_rest_api.hello_api.id  
  resource_id   = aws_api_gateway_resource.hello_resource.id  
  http_method   = "GET"  
  authorization = "NONE"  
}

# 4. Integrate API Gateway with Lambda  
resource "aws_api_gateway_integration" "hello_integration" {  
  rest_api_id             = aws_api_gateway_rest_api.hello_api.id  
  resource_id             = aws_api_gateway_resource.hello_resource.id  
  http_method             = aws_api_gateway_method.hello_method.http_method  
  integration_http_method = "POST" # API Gateway calls Lambda via POST  
  type                    = "AWS_PROXY"  
  uri                     = aws_lambda_function.hello_world.invoke_arn  
}

# 5. Grant API Gateway permission to invoke the Lambda  
resource "aws_lambda_permission" "allow_api_gateway" {  
  statement_id  = "AllowExecutionFromAPIGateway"  
  action        = "lambda:InvokeFunction"  
  function_name = aws_lambda_function.hello_world.function_name  
  principal     = "apigateway.amazonaws.com"  
  source_arn    = "${aws_api_gateway_rest_api.hello_api.execution_arn}/*/*"  
}

# 6. Deployment and Stage  
resource "aws_api_gateway_deployment" "hello_deployment" {  
  depends_on  = [aws_api_gateway_integration.hello_integration]  
  rest_api_id = aws_api_gateway_rest_api.hello_api.id  
}

resource "aws_api_gateway_stage" "hello_stage" {  
  deployment_id = aws_api_gateway_deployment.hello_deployment.id  
  rest_api_id   = aws_api_gateway_rest_api.hello_api.id  
  stage_name    = "v1"  
}

# Output the final URL  
output "api_endpoint" {  
  value = "https://${aws_api_gateway_rest_api.hello_api.id}[.execute-api.eu-central-1.amazonaws.com/v1/hello](https://.execute-api.eu-central-1.amazonaws.com/v1/hello)"  
}
```

## ---

**🚀 Step 4: Execution & Testing**

1. **Initialize:** Run terraform init to download the providers.  
2. **Apply:** Run terraform apply and type yes when prompted.  
3. **Verify:** Copy the api_endpoint URL from the output and test it via curl:  
   Bash  
   curl <your-api-endpoint>

## ---

**🏗️ Step 5: Refactor into a Reusable Module**

In production, you don't want to copy-paste this code for every new endpoint. We will move the logic into a module folder so you can deploy multiple APIs with a single block of code.

### **1. Create the Module Directory**

Bash

```bash
mkdir -p modules/serverless-api
```

### **2. Move Logic to the Module**

Create the following three files inside modules/serverless-api/.

**File: modules/serverless-api/variables.tf**

Define what parts of the API should be customizable.

Terraform

```hcl
variable "function_name" { type = string }  
variable "api_name"      { type = string }  
variable "path_part"     { type = string }  
variable "source_file"   { type = string }
```

**File: modules/serverless-api/main.tf**

Move the logic from your root file here, replacing hardcoded names with the variables defined above (e.g., name = var.api_name).

Terraform

```hcl
data "archive_file" "lambda_zip" {  
  type        = "zip"  
  source_file = var.source_file  
  output_path = "lambda_output/${var.function_name}.zip"  
}

resource "aws_iam_role" "lambda_exec" {  
  name = "${var.function_name}_role"  
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

resource "aws_lambda_function" "lambda" {  
  function_name = var.function_name  
  runtime       = "python3.9"  
  role          = aws_iam_role.lambda_exec.arn  
  handler       = "lambda_function.handler"  
  filename         = data.archive_file.lambda_zip.output_path  
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256  
}

resource "aws_api_gateway_rest_api" "api" {  
  name = var.api_name  
}

resource "aws_api_gateway_resource" "res" {  
  rest_api_id = aws_api_gateway_rest_api.api.id  
  parent_id   = aws_api_gateway_rest_api.api.root_resource_id  
  path_part   = var.path_part  
}

resource "aws_api_gateway_method" "method" {  
  rest_api_id   = aws_api_gateway_rest_api.api.id  
  resource_id   = aws_api_gateway_resource.res.id  
  http_method   = "GET"  
  authorization = "NONE"  
}

resource "aws_api_gateway_integration" "int" {  
  rest_api_id             = aws_api_gateway_rest_api.api.id  
  resource_id             = aws_api_gateway_resource.res.id  
  http_method             = aws_api_gateway_method.method.http_method  
  integration_http_method = "POST"  
  type                    = "AWS_PROXY"  
  uri                     = aws_lambda_function.lambda.invoke_arn  
}

resource "aws_lambda_permission" "allow_api" {  
  action        = "lambda:InvokeFunction"  
  function_name = aws_lambda_function.lambda.function_name  
  principal     = "apigateway.amazonaws.com"  
  source_arn    = "${aws_api_gateway_rest_api.api.execution_arn}/*/*"  
}

resource "aws_api_gateway_deployment" "dep" {  
  depends_on  = [aws_api_gateway_integration.int]  
  rest_api_id = aws_api_gateway_rest_api.api.id  
}

resource "aws_api_gateway_stage" "stage" {  
  deployment_id = aws_api_gateway_deployment.dep.id  
  rest_api_id   = aws_api_gateway_rest_api.api.id  
  stage_name    = "prod"  
}
```
**File: modules/serverless-api/outputs.tf**

Terraform

```hcl
output "endpoint" {  
  value = "https://${aws_api_gateway_rest_api.api.id}[.execute-api.eu-central-1.amazonaws.com/prod/$](https://.execute-api.eu-central-1.amazonaws.com/prod/$){aws_api_gateway_resource.res.path_part}"  
}
```

### **3. Use the Module in your Root main.tf**

Delete the previous resource definitions in your root main.tf and replace them with this clean call:

Terraform

```hcl
module "my_hello_api" {  
  source        = "./modules/serverless-api"  
  function_name = "hello_world_v2"  
  api_name      = "HelloWorldAPI"  
  path_part     = "greet"  
  source_file   = "lambda/lambda_function.py"  
}

output "module_api_endpoint" {  
  value = module.my_hello_api.endpoint  
}
```

4. **Re-Initialize & Apply:** Run terraform init (to register the new module) and then terraform apply.

## ---

**🛠️ Troubleshooting Guide**

| Issue | Possible Cause | Solution |
| :---- | :---- | :---- |
| **502 Bad Gateway** | Malformed Lambda Response | Ensure Python returns a dictionary with statusCode and body. |
| **403 Forbidden** | Permissions Missing | Verify the aws_lambda_permission resource is present in HCL. |
| **Changes not reflecting** | ZIP not updating | Ensure source_code_hash is using the data source output. |

## ---

**☁️ Step 6: Configure Remote S3 Backend**
Currently, your terraform.tfstate is stored locally on your machine. This is dangerous for teams. You will now migrate this state to the S3 bucket you created in the earlier exercise.

In your root main.tf, update the terraform block:

Terraform
```hcl
terraform {
  required_providers {
    aws     = { source = "hashicorp/aws", version = "~> 6.0" }
    archive = { source = "hashicorp/archive", version = "2.7.1" }
  }

  backend "s3" {
    bucket = "YOUR_S3_BUCKET_NAME_HERE" # Use the bucket from your previous lab
    key    = "serverless-lab/terraform.tfstate"
    region = "eu-central-1"
  }
}
```
Initialize Migration: Run the following command:

Bash
```hcl
terraform init -migrate-state
```
Confirm: When prompted to copy existing state to the new backend, type yes. Terraform will upload your local state to S3 and you can now safely delete your local .tfstate files.


**🧹 Step 7: Cleanup**

To avoid costs, destroy your infrastructure once finished:

Bash

```bash
terraform destroy
```
