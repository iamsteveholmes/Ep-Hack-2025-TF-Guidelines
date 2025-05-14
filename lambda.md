# AWS Lambda Terraform Best Practices

This document outlines the best practices for deploying and managing AWS Lambda functions using Terraform. Following these guidelines will help ensure your Lambda functions are secure, maintainable, and cost-effective.

## Table of Contents

- [Basic Lambda Configuration](#basic-lambda-configuration)
- [Deployment Strategies](#deployment-strategies)
- [Security Best Practices](#security-best-practices)
- [Performance Optimization](#performance-optimization)
- [Monitoring and Observability](#monitoring-and-observability)
- [Cost Optimization](#cost-optimization)
- [Integration with API Gateway](#integration-with-api-gateway)
- [Common Patterns and Examples](#common-patterns-and-examples)

## Basic Lambda Configuration

### Lambda Function Resource

```hcl
resource "aws_lambda_function" "example_function" {
  # Deployment package
  filename         = "lambda_function_payload.zip"  # Local deployment package
  # Alternative: Use S3 for larger packages
  # s3_bucket       = "my-lambda-deployments"
  # s3_key          = "functions/example-function/v1.0.0.zip"
  # s3_object_version = "1"

  # Function configuration
  function_name    = "${var.project_name}-${var.environment}-example-function"
  description      = "Example Lambda function for processing events"
  role             = aws_iam_role.lambda_execution_role.arn
  handler          = "index.handler"
  runtime          = "nodejs18.x"

  # Resource allocation
  memory_size      = 128  # MB, minimum is 128
  timeout          = 30   # seconds, maximum is 900 (15 minutes)

  # Environment variables - avoid storing secrets here
  environment {
    variables = {
      ENV             = var.environment
      LOG_LEVEL       = "INFO"
      REGION          = var.aws_region
      DYNAMODB_TABLE  = aws_dynamodb_table.example.name
    }
  }

  # Dead letter queue for failed executions
  dead_letter_config {
    target_arn = aws_sqs_queue.lambda_dlq.arn
  }

  # VPC configuration if needed
  vpc_config {
    subnet_ids         = var.private_subnet_ids
    security_group_ids = [aws_security_group.lambda_sg.id]
  }

  # Tracing
  tracing_config {
    mode = "Active"  # Enables AWS X-Ray tracing
  }

  # Tags for resource management
  tags = {
    Environment = var.environment
    Project     = var.project_name
    Function    = "example-function"
    ManagedBy   = "terraform"
  }
}
```

### Lambda Alias and Versioning

```hcl
# Publish a version from the latest Lambda code
resource "aws_lambda_function_version" "example_version" {
  function_name = aws_lambda_function.example_function.function_name
  description   = "Version ${var.app_version}"

  # Optional: Specify a specific S3 object version for the deployment package
  # s3_object_version = var.code_version

  # Ensure version is updated when function code changes
  depends_on = [
    aws_lambda_function.example_function
  ]

  # Prevent Terraform from redeploying when only the description changes
  lifecycle {
    ignore_changes = [
      description
    ]
  }
}

# Create an alias that points to the latest version
resource "aws_lambda_alias" "example_alias" {
  name             = var.environment
  description      = "Alias for ${var.environment} environment"
  function_name    = aws_lambda_function.example_function.function_name
  function_version = aws_lambda_function_version.example_version.version

  # Optional: Configure routing for canary deployments
  # routing_config {
  #   additional_version_weights = {
  #     "2" = 0.05  # Route 5% of traffic to version 2
  #   }
  # }
}
```

### Lambda Permissions

```hcl
# Allow API Gateway to invoke the Lambda function
resource "aws_lambda_permission" "api_gateway" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.example_function.function_name
  principal     = "apigateway.amazonaws.com"

  # Optionally restrict to specific API Gateway
  source_arn = "${aws_api_gateway_rest_api.example.execution_arn}/*/*"

  # Use the alias ARN to ensure permissions apply to the alias
  qualifier = aws_lambda_alias.example_alias.name
}
```

## Deployment Strategies

### Package Management

1. **Small Packages**: Keep deployment packages as small as possible to reduce cold start times.
2. **Layer Usage**: Use Lambda Layers for common dependencies:

```hcl
resource "aws_lambda_layer_version" "common_layer" {
  layer_name = "${var.project_name}-common-layer"
  description = "Common dependencies for Lambda functions"

  filename = "common_layer.zip"
  # Or use S3
  # s3_bucket = aws_s3_bucket.lambda_bucket.id
  # s3_key    = "layers/common_layer.zip"

  compatible_runtimes = ["nodejs18.x"]
}

# Reference the layer in your Lambda function
resource "aws_lambda_function" "with_layer" {
  # ... other configuration ...
  layers = [aws_lambda_layer_version.common_layer.arn]
}
```

### Blue/Green Deployments

Use Lambda aliases with traffic shifting for safe deployments:

```hcl
resource "aws_lambda_alias" "live" {
  name             = "live"
  function_name    = aws_lambda_function.example_function.function_name
  function_version = aws_lambda_function_version.current.version

  routing_config {
    additional_version_weights = {
      "${aws_lambda_function_version.new.version}" = 0.05  # Start with 5% traffic
    }
  }
}
```

## Security Best Practices

### IAM Role Configuration

Always follow the principle of least privilege:

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

# Create specific, limited policies instead of using managed policies
resource "aws_iam_policy" "lambda_permissions" {
  name        = "${var.project_name}-${var.environment}-lambda-policy"
  description = "IAM policy for Lambda function"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Effect   = "Allow"
        Resource = "arn:aws:logs:*:*:*"
      },
      {
        Action = [
          "dynamodb:GetItem",
          "dynamodb:PutItem"
        ]
        Effect   = "Allow"
        Resource = aws_dynamodb_table.example.arn
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_policy_attachment" {
  role       = aws_iam_role.lambda_execution_role.name
  policy_arn = aws_iam_policy.lambda_permissions.arn
}
```

### Secret Management

Use AWS Secrets Manager or Parameter Store for secrets, not environment variables:

```hcl
# Grant Lambda access to specific secrets
resource "aws_iam_policy" "secrets_access" {
  name        = "${var.project_name}-${var.environment}-secrets-policy"
  description = "Allow Lambda to access specific secrets"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = [
        "secretsmanager:GetSecretValue"
      ]
      Effect   = "Allow"
      Resource = aws_secretsmanager_secret.example.arn
    }]
  })
}

resource "aws_iam_role_policy_attachment" "secrets_policy_attachment" {
  role       = aws_iam_role.lambda_execution_role.name
  policy_arn = aws_iam_policy.secrets_access.arn
}
```

## Performance Optimization

### Memory and Timeout Configuration

```hcl
resource "aws_lambda_function" "optimized_function" {
  # ... other configuration ...

  # Allocate appropriate memory based on function needs
  # More memory also means more CPU allocation
  memory_size = 1024  # MB

  # Set timeout based on expected execution time plus buffer
  timeout     = 60    # seconds
}
```

### Provisioned Concurrency

For functions that need to avoid cold starts:

```hcl
resource "aws_lambda_provisioned_concurrency_config" "example" {
  function_name                     = aws_lambda_function.example_function.function_name
  provisioned_concurrent_executions = 5
  qualifier                         = aws_lambda_alias.example_alias.name
}
```

## Monitoring and Observability

### CloudWatch Logs

```hcl
# Log group with appropriate retention
resource "aws_cloudwatch_log_group" "lambda_logs" {
  name              = "/aws/lambda/${aws_lambda_function.example_function.function_name}"
  retention_in_days = 30

  tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}
```

### CloudWatch Alarms

```hcl
# Alarm for errors
resource "aws_cloudwatch_metric_alarm" "lambda_errors" {
  alarm_name          = "${aws_lambda_function.example_function.function_name}-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  period              = 60
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "This metric monitors lambda function errors"

  dimensions = {
    FunctionName = aws_lambda_function.example_function.function_name
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}

# Alarm for throttles
resource "aws_cloudwatch_metric_alarm" "lambda_throttles" {
  alarm_name          = "${aws_lambda_function.example_function.function_name}-throttles"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "Throttles"
  namespace           = "AWS/Lambda"
  period              = 60
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "This metric monitors lambda function throttles"

  dimensions = {
    FunctionName = aws_lambda_function.example_function.function_name
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

### X-Ray Tracing

```hcl
# Enable X-Ray tracing
resource "aws_lambda_function" "traced_function" {
  # ... other configuration ...

  tracing_config {
    mode = "Active"
  }
}

# Add X-Ray permissions to the Lambda role
resource "aws_iam_policy" "xray_policy" {
  name        = "${var.project_name}-${var.environment}-xray-policy"
  description = "Allow Lambda to send traces to X-Ray"

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

resource "aws_iam_role_policy_attachment" "xray_policy_attachment" {
  role       = aws_iam_role.lambda_execution_role.name
  policy_arn = aws_iam_policy.xray_policy.arn
}
```

## Cost Optimization

1. **Right-size Memory**: Allocate only the memory your function needs
2. **Optimize Code**: Reduce execution time to save on compute costs
3. **Use Reserved Concurrency**: Limit maximum concurrency to control costs

```hcl
# Set reserved concurrency to limit maximum invocations
resource "aws_lambda_function" "cost_optimized" {
  # ... other configuration ...

  reserved_concurrent_executions = 10  # Limit to 10 concurrent executions
}
```

## Integration with API Gateway

See the [API Gateway Guidelines](api_gateway.md) for detailed integration patterns.

Basic integration example:

```hcl
# Lambda function for API
resource "aws_lambda_function" "api_function" {
  # ... configuration ...
}

# Permission for API Gateway to invoke Lambda
resource "aws_lambda_permission" "api_gateway_permission" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.api_function.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_api_gateway_rest_api.example.execution_arn}/*/*"
}
```

## Common Patterns and Examples

### Event Source Mapping

```hcl
# DynamoDB Stream to Lambda
resource "aws_lambda_event_source_mapping" "dynamodb_stream" {
  event_source_arn  = aws_dynamodb_table.example.stream_arn
  function_name     = aws_lambda_function.example_function.arn
  starting_position = "LATEST"

  # Optional: Filter events
  filter_criteria {
    filter {
      pattern = jsonencode({
        eventName: ["INSERT", "MODIFY"]
      })
    }
  }
}

# SQS Queue to Lambda
resource "aws_lambda_event_source_mapping" "sqs_event_source" {
  event_source_arn = aws_sqs_queue.example.arn
  function_name    = aws_lambda_function.example_function.arn
  batch_size       = 10

  # Enable scaling up to 5 concurrent batches
  scaling_config {
    maximum_concurrency = 5
  }
}
```

### Lambda as a Step Function Task

```hcl
resource "aws_sfn_state_machine" "example_workflow" {
  name     = "${var.project_name}-${var.environment}-workflow"
  role_arn = aws_iam_role.step_function_role.arn

  definition = <<EOF
{
  "Comment": "A simple AWS Step Functions state machine",
  "StartAt": "ProcessData",
  "States": {
    "ProcessData": {
      "Type": "Task",
      "Resource": "${aws_lambda_function.example_function.arn}",
      "End": true
    }
  }
}
EOF
}
```

### Lambda Function URL

```hcl
resource "aws_lambda_function_url" "example_url" {
  function_name      = aws_lambda_function.example_function.function_name
  authorization_type = "AWS_IAM"  # Or "NONE" for public access

  # Optional: Use the alias instead of $LATEST
  qualifier = aws_lambda_alias.example_alias.name
}

output "function_url" {
  value = aws_lambda_function_url.example_url.function_url
}
```

---

For more information on integrating Lambda with other AWS services, refer to the other documents in this repository:
- [API Gateway Integration](api_gateway.md)
- [IAM Best Practices](iam.md)
- [Monitoring and Observability](monitoring.md)
