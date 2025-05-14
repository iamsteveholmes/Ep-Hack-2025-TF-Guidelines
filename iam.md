---
title: "IAM Best Practices for AWS API Gateway and Lambda with Terraform"
category: "Security"
tags: ["aws", "iam", "api gateway", "lambda", "terraform", "security", "best practices"]
last_updated: "2025-05-14"
---

# IAM Best Practices for AWS API Gateway and Lambda with Terraform

This document outlines the best practices for creating and managing IAM roles and policies for AWS API Gateway and Lambda using Terraform. Following these guidelines will help ensure your serverless applications are secure and follow the principle of least privilege.

## Table of Contents

- [General IAM Best Practices](#general-iam-best-practices)
- [Lambda Execution Roles](#lambda-execution-roles)
- [API Gateway IAM Roles](#api-gateway-iam-roles)
- [Cross-Service Permissions](#cross-service-permissions)
- [Resource-Based Policies](#resource-based-policies)
- [IAM Policy Templates](#iam-policy-templates)
- [Security Best Practices](#security-best-practices)
- [Monitoring and Auditing](#monitoring-and-auditing)

## General IAM Best Practices

### Principle of Least Privilege

Always grant the minimum permissions necessary for a resource to perform its function:

```hcl
# BAD: Overly permissive policy
resource "aws_iam_policy" "overly_permissive" {
  name        = "overly-permissive-policy"
  description = "Grants too many permissions"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = "dynamodb:*"  # Too broad
      Resource = "*"           # Too broad
    }]
  })
}

# GOOD: Properly scoped policy
resource "aws_iam_policy" "properly_scoped" {
  name        = "properly-scoped-policy"
  description = "Grants only necessary permissions"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query"
      ]
      Resource = aws_dynamodb_table.example.arn
    }]
  })
}
```

### Use IAM Roles Instead of IAM Users

For services like Lambda and API Gateway, always use IAM roles instead of IAM users:

```hcl
# Create a role for Lambda execution
resource "aws_iam_role" "lambda_execution_role" {
  name = "${var.project_name}-${var.environment}-lambda-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })

  # Optional: Add a description
  description = "IAM role for Lambda function execution"

  # Optional: Add path for organizational purposes
  path = "/service-roles/lambda/"

  # Optional: Add permission boundary
  # permissions_boundary = aws_iam_policy.boundary.arn

  tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}
```

### Use Managed Policies Sparingly

AWS managed policies are often too permissive. Create custom policies that grant only the permissions needed:

```hcl
# AVOID: Using AWS managed policy with broad permissions
resource "aws_iam_role_policy_attachment" "lambda_basic_execution" {
  role       = aws_iam_role.lambda_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/AWSLambdaBasicExecutionRole"
}

# BETTER: Create a custom policy with specific permissions
resource "aws_iam_policy" "lambda_logging" {
  name        = "${var.project_name}-${var.environment}-lambda-logging"
  description = "IAM policy for logging from a Lambda"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ]
      Effect   = "Allow"
      Resource = "arn:aws:logs:*:*:log-group:/aws/lambda/${var.function_name}:*"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_logs" {
  role       = aws_iam_role.lambda_execution_role.name
  policy_arn = aws_iam_policy.lambda_logging.arn
}
```

## Lambda Execution Roles

### Basic Lambda Execution Role

```hcl
resource "aws_iam_role" "lambda_execution_role" {
  name = "${var.project_name}-${var.environment}-lambda-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}

# Logging permissions
resource "aws_iam_policy" "lambda_logging" {
  name        = "${var.project_name}-${var.environment}-lambda-logging"
  description = "IAM policy for logging from a Lambda"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ]
      Effect   = "Allow"
      Resource = "arn:aws:logs:*:*:log-group:/aws/lambda/${var.function_name}:*"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_logs" {
  role       = aws_iam_role.lambda_execution_role.name
  policy_arn = aws_iam_policy.lambda_logging.arn
}
```

### Lambda with DynamoDB Access

```hcl
resource "aws_iam_policy" "lambda_dynamodb" {
  name        = "${var.project_name}-${var.environment}-lambda-dynamodb"
  description = "IAM policy for Lambda to access DynamoDB"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query",
        "dynamodb:Scan"
      ]
      Effect   = "Allow"
      Resource = [
        aws_dynamodb_table.example.arn,
        "${aws_dynamodb_table.example.arn}/index/*"
      ]
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_dynamodb" {
  role       = aws_iam_role.lambda_execution_role.name
  policy_arn = aws_iam_policy.lambda_dynamodb.arn
}
```

### Lambda with S3 Access

```hcl
resource "aws_iam_policy" "lambda_s3" {
  name        = "${var.project_name}-${var.environment}-lambda-s3"
  description = "IAM policy for Lambda to access S3"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "s3:GetObject",
          "s3:ListBucket"
        ]
        Effect   = "Allow"
        Resource = [
          aws_s3_bucket.example.arn,
          "${aws_s3_bucket.example.arn}/*"
        ]
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_s3" {
  role       = aws_iam_role.lambda_execution_role.name
  policy_arn = aws_iam_policy.lambda_s3.arn
}
```

### Lambda with VPC Access

```hcl
resource "aws_iam_policy" "lambda_vpc" {
  name        = "${var.project_name}-${var.environment}-lambda-vpc"
  description = "IAM policy for Lambda to access VPC"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = [
        "ec2:CreateNetworkInterface",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DeleteNetworkInterface",
        "ec2:AssignPrivateIpAddresses",
        "ec2:UnassignPrivateIpAddresses"
      ]
      Effect   = "Allow"
      Resource = "*"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_vpc" {
  role       = aws_iam_role.lambda_execution_role.name
  policy_arn = aws_iam_policy.lambda_vpc.arn
}
```

### Lambda with X-Ray Tracing

```hcl
resource "aws_iam_policy" "lambda_xray" {
  name        = "${var.project_name}-${var.environment}-lambda-xray"
  description = "IAM policy for Lambda to use X-Ray tracing"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = [
        "xray:PutTraceSegments",
        "xray:PutTelemetryRecords"
      ]
      Effect   = "Allow"
      Resource = "*"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_xray" {
  role       = aws_iam_role.lambda_execution_role.name
  policy_arn = aws_iam_policy.lambda_xray.arn
}
```

## API Gateway IAM Roles

### CloudWatch Logs Role for API Gateway

```hcl
resource "aws_iam_role" "api_gateway_cloudwatch" {
  name = "${var.project_name}-${var.environment}-api-cloudwatch-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "apigateway.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_policy" "api_gateway_logging" {
  name        = "${var.project_name}-${var.environment}-api-logging"
  description = "IAM policy for API Gateway to write logs to CloudWatch"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams",
        "logs:PutLogEvents",
        "logs:GetLogEvents",
        "logs:FilterLogEvents"
      ]
      Effect   = "Allow"
      Resource = "arn:aws:logs:*:*:*"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "api_gateway_logs" {
  role       = aws_iam_role.api_gateway_cloudwatch.name
  policy_arn = aws_iam_policy.api_gateway_logging.arn
}

# Set up account-level settings for API Gateway
resource "aws_api_gateway_account" "main" {
  cloudwatch_role_arn = aws_iam_role.api_gateway_cloudwatch.arn
}
```

### API Gateway Invocation Role for Lambda

```hcl
resource "aws_iam_role" "api_gateway_lambda_invocation" {
  name = "${var.project_name}-${var.environment}-api-lambda-invocation"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "apigateway.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_policy" "lambda_invocation" {
  name        = "${var.project_name}-${var.environment}-lambda-invocation"
  description = "IAM policy for API Gateway to invoke Lambda"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action   = "lambda:InvokeFunction"
      Effect   = "Allow"
      Resource = aws_lambda_function.example.arn
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_invocation" {
  role       = aws_iam_role.api_gateway_lambda_invocation.name
  policy_arn = aws_iam_policy.lambda_invocation.arn
}
```

## Cross-Service Permissions

### Lambda Invoking Other AWS Services

```hcl
resource "aws_iam_policy" "lambda_sns" {
  name        = "${var.project_name}-${var.environment}-lambda-sns"
  description = "IAM policy for Lambda to publish to SNS"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action   = "sns:Publish"
      Effect   = "Allow"
      Resource = aws_sns_topic.example.arn
    }]
  })
}

resource "aws_iam_policy" "lambda_sqs" {
  name        = "${var.project_name}-${var.environment}-lambda-sqs"
  description = "IAM policy for Lambda to send/receive SQS messages"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = [
        "sqs:SendMessage",
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ]
      Effect   = "Allow"
      Resource = aws_sqs_queue.example.arn
    }]
  })
}

resource "aws_iam_policy" "lambda_secrets_manager" {
  name        = "${var.project_name}-${var.environment}-lambda-secrets"
  description = "IAM policy for Lambda to access Secrets Manager"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action   = "secretsmanager:GetSecretValue"
      Effect   = "Allow"
      Resource = aws_secretsmanager_secret.example.arn
    }]
  })
}
```

### Cross-Account Access

```hcl
resource "aws_iam_role" "lambda_cross_account" {
  name = "${var.project_name}-${var.environment}-lambda-cross-account"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_policy" "assume_role_policy" {
  name        = "${var.project_name}-${var.environment}-assume-role"
  description = "IAM policy for Lambda to assume a role in another account"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action   = "sts:AssumeRole"
      Effect   = "Allow"
      Resource = "arn:aws:iam::${var.target_account_id}:role/${var.target_role_name}"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_assume_role" {
  role       = aws_iam_role.lambda_cross_account.name
  policy_arn = aws_iam_policy.assume_role_policy.arn
}
```

## Resource-Based Policies

### Lambda Permission for API Gateway

```hcl
resource "aws_lambda_permission" "api_gateway" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.example.function_name
  principal     = "apigateway.amazonaws.com"

  # Restrict to specific API Gateway
  source_arn = "${aws_api_gateway_rest_api.example.execution_arn}/*/*"
}
```

### Lambda Permission for S3 Event

```hcl
resource "aws_lambda_permission" "s3_trigger" {
  statement_id  = "AllowS3Invoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.example.function_name
  principal     = "s3.amazonaws.com"
  source_arn    = aws_s3_bucket.example.arn
}
```

### Lambda Permission for CloudWatch Events

```hcl
resource "aws_lambda_permission" "cloudwatch_events" {
  statement_id  = "AllowCloudWatchEventsInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.example.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.example.arn
}
```

## IAM Policy Templates

### Reusable Policy Module

Create a module for common IAM policies:

```hcl
# modules/iam_policies/main.tf

variable "project_name" {
  description = "Name of the project"
  type        = string
}

variable "environment" {
  description = "Deployment environment"
  type        = string
}

variable "resources" {
  description = "Map of resource ARNs"
  type        = map(string)
}

# DynamoDB read-only policy
resource "aws_iam_policy" "dynamodb_read" {
  name        = "${var.project_name}-${var.environment}-dynamodb-read"
  description = "IAM policy for read-only access to DynamoDB"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = [
        "dynamodb:GetItem",
        "dynamodb:BatchGetItem",
        "dynamodb:Query",
        "dynamodb:Scan"
      ]
      Effect   = "Allow"
      Resource = var.resources["dynamodb_table"]
    }]
  })
}

# DynamoDB read-write policy
resource "aws_iam_policy" "dynamodb_read_write" {
  name        = "${var.project_name}-${var.environment}-dynamodb-read-write"
  description = "IAM policy for read-write access to DynamoDB"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = [
        "dynamodb:GetItem",
        "dynamodb:BatchGetItem",
        "dynamodb:Query",
        "dynamodb:Scan",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:BatchWriteItem"
      ]
      Effect   = "Allow"
      Resource = var.resources["dynamodb_table"]
    }]
  })
}

# S3 read-only policy
resource "aws_iam_policy" "s3_read" {
  name        = "${var.project_name}-${var.environment}-s3-read"
  description = "IAM policy for read-only access to S3"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = [
        "s3:GetObject",
        "s3:ListBucket"
      ]
      Effect   = "Allow"
      Resource = [
        var.resources["s3_bucket"],
        "${var.resources["s3_bucket"]}/*"
      ]
    }]
  })
}

# Output policies for use in other modules
output "dynamodb_read_policy_arn" {
  value = aws_iam_policy.dynamodb_read.arn
}

output "dynamodb_read_write_policy_arn" {
  value = aws_iam_policy.dynamodb_read_write.arn
}

output "s3_read_policy_arn" {
  value = aws_iam_policy.s3_read.arn
}
```

Usage:

```hcl
module "iam_policies" {
  source = "./modules/iam_policies"

  project_name = var.project_name
  environment  = var.environment
  resources = {
    dynamodb_table = aws_dynamodb_table.example.arn
    s3_bucket      = aws_s3_bucket.example.arn
  }
}

resource "aws_iam_role_policy_attachment" "lambda_dynamodb_read" {
  role       = aws_iam_role.lambda_execution_role.name
  policy_arn = module.iam_policies.dynamodb_read_policy_arn
}
```

## Security Best Practices

### Use Condition Keys

```hcl
resource "aws_iam_policy" "s3_secure_transport" {
  name        = "${var.project_name}-${var.environment}-s3-secure-transport"
  description = "IAM policy for S3 access with secure transport"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action   = "s3:*"
      Effect   = "Allow"
      Resource = [
        aws_s3_bucket.example.arn,
        "${aws_s3_bucket.example.arn}/*"
      ]
      Condition = {
        Bool = {
          "aws:SecureTransport": "true"
        }
      }
    }]
  })
}
```

### Implement Permission Boundaries

```hcl
resource "aws_iam_policy" "permission_boundary" {
  name        = "${var.project_name}-permission-boundary"
  description = "Permission boundary for all roles in the project"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = [
          "s3:*",
          "dynamodb:*",
          "lambda:*",
          "logs:*",
          "apigateway:*"
        ]
        Resource = "*"
      },
      {
        Effect   = "Deny"
        Action   = [
          "iam:CreateUser",
          "iam:CreateRole",
          "iam:DeleteRole",
          "ec2:*",
          "rds:*"
        ]
        Resource = "*"
      }
    ]
  })
}

resource "aws_iam_role" "lambda_with_boundary" {
  name                 = "${var.project_name}-${var.environment}-lambda-role"
  assume_role_policy   = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
  permissions_boundary = aws_iam_policy.permission_boundary.arn
}
```

### Regularly Rotate Credentials

For IAM users (when necessary):

```hcl
resource "aws_iam_user" "api_user" {
  name = "${var.project_name}-${var.environment}-api-user"
  path = "/system/"
}

resource "aws_iam_access_key" "api_user" {
  user = aws_iam_user.api_user.name

  # Set to true to deactivate the access key
  # status = "Inactive"

  # Use lifecycle to enforce rotation
  lifecycle {
    create_before_destroy = true
  }
}

# Store access key securely in Secrets Manager
resource "aws_secretsmanager_secret" "api_credentials" {
  name = "${var.project_name}/${var.environment}/api-credentials"

  # Automatically rotate every 90 days
  rotation_lambda_arn = aws_lambda_function.rotate_credentials.arn
  rotation_rules {
    automatically_after_days = 90
  }
}

resource "aws_secretsmanager_secret_version" "api_credentials" {
  secret_id     = aws_secretsmanager_secret.api_credentials.id
  secret_string = jsonencode({
    access_key = aws_iam_access_key.api_user.id
    secret_key = aws_iam_access_key.api_user.secret
  })
}
```

## Monitoring and Auditing

### Enable CloudTrail

```hcl
resource "aws_cloudtrail" "main" {
  name                          = "${var.project_name}-${var.environment}-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true

  event_selector {
    read_write_type           = "All"
    include_management_events = true

    data_resource {
      type   = "AWS::Lambda::Function"
      values = ["arn:aws:lambda"]
    }

    data_resource {
      type   = "AWS::S3::Object"
      values = ["${aws_s3_bucket.example.arn}/"]
    }
  }
}
```

### IAM Access Analyzer

```hcl
resource "aws_accessanalyzer_analyzer" "main" {
  analyzer_name = "${var.project_name}-${var.environment}-analyzer"
  type          = "ACCOUNT"  # or "ORGANIZATION" for org-wide analysis
}
```

### AWS Config Rules for IAM

```hcl
resource "aws_config_config_rule" "iam_user_no_policies_check" {
  name        = "iam-user-no-policies-check"
  description = "Checks that IAM users do not have policies attached"

  source {
    owner             = "AWS"
    source_identifier = "IAM_USER_NO_POLICIES_CHECK"
  }
}

resource "aws_config_config_rule" "iam_root_access_key_check" {
  name        = "iam-root-access-key-check"
  description = "Checks that the root user does not have access keys"

  source {
    owner             = "AWS"
    source_identifier = "IAM_ROOT_ACCESS_KEY_CHECK"
  }
}
```

---

For more information on integrating IAM with other AWS services, refer to the other documents in this repository:
- [API Gateway Guidelines](api_gateway.md)
- [Lambda Guidelines](lambda.md)
- [Provider Configuration](provider.md)
