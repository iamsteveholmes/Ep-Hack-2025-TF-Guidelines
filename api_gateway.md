---
title: "AWS API Gateway V1 (REST API) Terraform Best Practices"
category: "API Gateway"
tags: ["aws", "api gateway", "terraform", "rest api", "best practices"]
last_updated: "2025-05-14"
---

# AWS API Gateway V1 (REST API) Terraform Best Practices

This document outlines the best practices for deploying and managing AWS API Gateway REST APIs (API Gateway V1) using Terraform. Following these guidelines will help ensure your API Gateway deployments are secure, maintainable, and cost-effective.

## Table of Contents

- [Basic API Gateway Configuration](#basic-api-gateway-configuration)
- [Resource and Method Configuration](#resource-and-method-configuration)
- [Integration with Lambda](#integration-with-lambda)
- [Authorization and Authentication](#authorization-and-authentication)
- [Request and Response Handling](#request-and-response-handling)
- [Deployment Strategies](#deployment-strategies)
- [Monitoring and Logging](#monitoring-and-logging)
- [Performance Optimization](#performance-optimization)
- [Security Best Practices](#security-best-practices)
- [Cost Optimization](#cost-optimization)
- [Common Patterns and Examples](#common-patterns-and-examples)

## Basic API Gateway Configuration

### REST API Resource

```hcl
resource "aws_api_gateway_rest_api" "example_api" {
  name        = "${var.project_name}-${var.environment}-api"
  description = "Example REST API for ${var.project_name}"

  # Enable binary media types if needed
  binary_media_types = [
    "application/octet-stream",
    "image/jpeg"
  ]

  # API Gateway endpoint configuration
  endpoint_configuration {
    types = ["REGIONAL"]  # Options: EDGE, REGIONAL, PRIVATE
  }

  # Disable default endpoint if using custom domain
  # disable_execute_api_endpoint = true

  # Tags for resource management
  tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "terraform"
  }
}
```

### API Gateway Domain Name

```hcl
resource "aws_api_gateway_domain_name" "example" {
  domain_name              = "api.example.com"
  regional_certificate_arn = aws_acm_certificate.example.arn

  endpoint_configuration {
    types = ["REGIONAL"]
  }

  tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}

# Map domain name to API stage
resource "aws_api_gateway_base_path_mapping" "example" {
  api_id      = aws_api_gateway_rest_api.example_api.id
  stage_name  = aws_api_gateway_stage.example.stage_name
  domain_name = aws_api_gateway_domain_name.example.domain_name
  base_path   = "v1"  # Use "" for root path
}
```

## Resource and Method Configuration

### API Resources

```hcl
# Root resource is created automatically with the REST API
# Create child resources for your API paths

# /items resource
resource "aws_api_gateway_resource" "items" {
  rest_api_id = aws_api_gateway_rest_api.example_api.id
  parent_id   = aws_api_gateway_rest_api.example_api.root_resource_id
  path_part   = "items"
}

# /items/{itemId} resource
resource "aws_api_gateway_resource" "item" {
  rest_api_id = aws_api_gateway_rest_api.example_api.id
  parent_id   = aws_api_gateway_resource.items.id
  path_part   = "{itemId}"
}
```

### Method Configuration

```hcl
# GET method on /items
resource "aws_api_gateway_method" "get_items" {
  rest_api_id   = aws_api_gateway_rest_api.example_api.id
  resource_id   = aws_api_gateway_resource.items.id
  http_method   = "GET"
  authorization_type = "COGNITO_USER_POOLS"  # Options: NONE, AWS_IAM, CUSTOM, COGNITO_USER_POOLS
  authorizer_id = aws_api_gateway_authorizer.cognito.id

  # API key requirement
  api_key_required = false

  # Request validator
  request_validator_id = aws_api_gateway_request_validator.validator.id

  # Request parameters (query string, path, header)
  request_parameters = {
    "method.request.querystring.limit" = false  # false = optional, true = required
    "method.request.querystring.page"  = false
  }
}

# POST method on /items
resource "aws_api_gateway_method" "post_item" {
  rest_api_id   = aws_api_gateway_rest_api.example_api.id
  resource_id   = aws_api_gateway_resource.items.id
  http_method   = "POST"
  authorization_type = "AWS_IAM"

  # Request validator
  request_validator_id = aws_api_gateway_request_validator.validator.id

  # Request models for validation
  request_models = {
    "application/json" = aws_api_gateway_model.item_model.name
  }
}
```

### Request Validation

```hcl
# Request validator
resource "aws_api_gateway_request_validator" "validator" {
  name                        = "example-validator"
  rest_api_id                 = aws_api_gateway_rest_api.example_api.id
  validate_request_body       = true
  validate_request_parameters = true
}

# Model for request validation
resource "aws_api_gateway_model" "item_model" {
  rest_api_id  = aws_api_gateway_rest_api.example_api.id
  name         = "ItemModel"
  description  = "JSON Schema for Item"
  content_type = "application/json"

  schema = jsonencode({
    "$schema" = "http://json-schema.org/draft-04/schema#"
    title     = "ItemModel"
    type      = "object"
    required  = ["name", "description"]
    properties = {
      name = {
        type = "string"
      }
      description = {
        type = "string"
      }
      price = {
        type = "number"
      }
    }
  })
}
```

## Integration with Lambda

### Lambda Integration

```hcl
# Integration with Lambda for GET /items
resource "aws_api_gateway_integration" "get_items_lambda" {
  rest_api_id             = aws_api_gateway_rest_api.example_api.id
  resource_id             = aws_api_gateway_resource.items.id
  http_method             = aws_api_gateway_method.get_items.http_method
  integration_http_method = "POST"  # Always POST for Lambda integrations
  type                    = "AWS_PROXY"  # Lambda proxy integration
  uri                     = aws_lambda_function.get_items.invoke_arn

  # For non-proxy integrations, you can specify request templates
  # request_templates = {
  #   "application/json" = file("${path.module}/templates/request_template.vtl")
  # }
}

# Lambda permission to allow API Gateway to invoke the function
resource "aws_lambda_permission" "api_gateway_get_items" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.get_items.function_name
  principal     = "apigateway.amazonaws.com"

  # Restrict to this specific API Gateway method
  source_arn = "${aws_api_gateway_rest_api.example_api.execution_arn}/*/${aws_api_gateway_method.get_items.http_method}${aws_api_gateway_resource.items.path}"
}
```

### Method Response and Integration Response

```hcl
# Method response (for non-proxy integrations)
resource "aws_api_gateway_method_response" "get_items_200" {
  rest_api_id = aws_api_gateway_rest_api.example_api.id
  resource_id = aws_api_gateway_resource.items.id
  http_method = aws_api_gateway_method.get_items.http_method
  status_code = "200"

  response_models = {
    "application/json" = "Empty"
  }

  response_parameters = {
    "method.response.header.Access-Control-Allow-Origin" = true
  }
}

# Integration response (for non-proxy integrations)
resource "aws_api_gateway_integration_response" "get_items_integration_response" {
  rest_api_id = aws_api_gateway_rest_api.example_api.id
  resource_id = aws_api_gateway_resource.items.id
  http_method = aws_api_gateway_method.get_items.http_method
  status_code = aws_api_gateway_method_response.get_items_200.status_code

  # Response templates for transforming the Lambda output
  response_templates = {
    "application/json" = file("${path.module}/templates/response_template.vtl")
  }

  response_parameters = {
    "method.response.header.Access-Control-Allow-Origin" = "'*'"
  }
}
```

## Authorization and Authentication

### Cognito User Pool Authorizer

```hcl
resource "aws_api_gateway_authorizer" "cognito" {
  name                   = "cognito-authorizer"
  rest_api_id            = aws_api_gateway_rest_api.example_api.id
  type                   = "COGNITO_USER_POOLS"
  provider_arns          = [aws_cognito_user_pool.example.arn]
  identity_source        = "method.request.header.Authorization"
  authorizer_result_ttl_in_seconds = 300
}
```

### Lambda Authorizer (Custom)

```hcl
resource "aws_api_gateway_authorizer" "custom" {
  name                   = "custom-authorizer"
  rest_api_id            = aws_api_gateway_rest_api.example_api.id
  authorizer_uri         = aws_lambda_function.authorizer.invoke_arn
  authorizer_credentials = aws_iam_role.invocation_role.arn
  type                   = "TOKEN"  # or "REQUEST" for request parameter-based
  identity_source        = "method.request.header.Authorization"
  authorizer_result_ttl_in_seconds = 300
}

# IAM role for the API Gateway to invoke the authorizer Lambda
resource "aws_iam_role" "invocation_role" {
  name = "api_gateway_auth_invocation"

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

resource "aws_iam_role_policy" "invocation_policy" {
  name = "default"
  role = aws_iam_role.invocation_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action   = "lambda:InvokeFunction"
      Effect   = "Allow"
      Resource = aws_lambda_function.authorizer.arn
    }]
  })
}
```

### API Key and Usage Plans

```hcl
# API key
resource "aws_api_gateway_api_key" "example" {
  name        = "${var.project_name}-${var.environment}-key"
  description = "API key for ${var.project_name}"
  enabled     = true
}

# Usage plan
resource "aws_api_gateway_usage_plan" "example" {
  name        = "${var.project_name}-${var.environment}-usage-plan"
  description = "Usage plan for ${var.project_name}"

  api_stages {
    api_id = aws_api_gateway_rest_api.example_api.id
    stage  = aws_api_gateway_stage.example.stage_name
  }

  quota_settings {
    limit  = 1000
    offset = 0
    period = "MONTH"
  }

  throttle_settings {
    burst_limit = 20
    rate_limit  = 10
  }
}

# Associate API key with usage plan
resource "aws_api_gateway_usage_plan_key" "example" {
  key_id        = aws_api_gateway_api_key.example.id
  key_type      = "API_KEY"
  usage_plan_id = aws_api_gateway_usage_plan.example.id
}
```

## Request and Response Handling

### CORS Configuration

```hcl
# OPTIONS method for CORS
resource "aws_api_gateway_method" "options_items" {
  rest_api_id   = aws_api_gateway_rest_api.example_api.id
  resource_id   = aws_api_gateway_resource.items.id
  http_method   = "OPTIONS"
  authorization_type = "NONE"
}

# Mock integration for OPTIONS
resource "aws_api_gateway_integration" "options_integration" {
  rest_api_id = aws_api_gateway_rest_api.example_api.id
  resource_id = aws_api_gateway_resource.items.id
  http_method = aws_api_gateway_method.options_items.http_method
  type        = "MOCK"

  request_templates = {
    "application/json" = jsonencode({
      statusCode = 200
    })
  }
}

# Method response for OPTIONS
resource "aws_api_gateway_method_response" "options_200" {
  rest_api_id = aws_api_gateway_rest_api.example_api.id
  resource_id = aws_api_gateway_resource.items.id
  http_method = aws_api_gateway_method.options_items.http_method
  status_code = "200"

  response_models = {
    "application/json" = "Empty"
  }

  response_parameters = {
    "method.response.header.Access-Control-Allow-Headers" = true
    "method.response.header.Access-Control-Allow-Methods" = true
    "method.response.header.Access-Control-Allow-Origin"  = true
  }
}

# Integration response for OPTIONS
resource "aws_api_gateway_integration_response" "options_integration_response" {
  rest_api_id = aws_api_gateway_rest_api.example_api.id
  resource_id = aws_api_gateway_resource.items.id
  http_method = aws_api_gateway_method.options_items.http_method
  status_code = aws_api_gateway_method_response.options_200.status_code

  response_parameters = {
    "method.response.header.Access-Control-Allow-Headers" = "'Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token'"
    "method.response.header.Access-Control-Allow-Methods" = "'GET,POST,OPTIONS'"
    "method.response.header.Access-Control-Allow-Origin"  = "'*'"
  }
}
```

### Gateway Responses (Global Error Handling)

```hcl
# Customize 4xx responses
resource "aws_api_gateway_gateway_response" "unauthorized" {
  rest_api_id   = aws_api_gateway_rest_api.example_api.id
  response_type = "UNAUTHORIZED"
  status_code   = "401"

  response_templates = {
    "application/json" = jsonencode({
      message = "Unauthorized access"
      error   = "Access denied due to missing or invalid authentication"
    })
  }

  response_parameters = {
    "gatewayresponse.header.Access-Control-Allow-Origin" = "'*'"
  }
}

# Customize 5xx responses
resource "aws_api_gateway_gateway_response" "server_error" {
  rest_api_id   = aws_api_gateway_rest_api.example_api.id
  response_type = "DEFAULT_5XX"

  response_templates = {
    "application/json" = jsonencode({
      message = "Internal server error"
      error   = "An unexpected error occurred"
    })
  }

  response_parameters = {
    "gatewayresponse.header.Access-Control-Allow-Origin" = "'*'"
  }
}
```

## Deployment Strategies

### API Gateway Deployment

```hcl
resource "aws_api_gateway_deployment" "example" {
  rest_api_id = aws_api_gateway_rest_api.example_api.id

  # Ensure deployment happens after all resources are created
  depends_on = [
    aws_api_gateway_integration.get_items_lambda,
    aws_api_gateway_integration.post_item_lambda,
    aws_api_gateway_integration.options_integration
  ]

  # Use a trigger to force redeployment when needed
  triggers = {
    redeployment = sha1(jsonencode([
      aws_api_gateway_resource.items.id,
      aws_api_gateway_resource.item.id,
      aws_api_gateway_method.get_items.id,
      aws_api_gateway_method.post_item.id,
      aws_api_gateway_integration.get_items_lambda.id,
      # Add other resources that should trigger redeployment
    ]))
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

### API Gateway Stage

```hcl
resource "aws_api_gateway_stage" "example" {
  deployment_id = aws_api_gateway_deployment.example.id
  rest_api_id   = aws_api_gateway_rest_api.example_api.id
  stage_name    = var.environment
  description   = "${var.environment} stage for ${var.project_name}"

  # Cache settings
  cache_cluster_enabled = true
  cache_cluster_size    = "0.5"  # Valid values: 0.5, 1.6, 6.1, 13.5, 28.4, 58.2, 118, 237

  # Enable CloudWatch logs
  access_log_settings {
    destination_arn = aws_cloudwatch_log_group.api_gateway_logs.arn
    format = jsonencode({
      requestId      = "$context.requestId"
      ip             = "$context.identity.sourceIp"
      caller         = "$context.identity.caller"
      user           = "$context.identity.user"
      requestTime    = "$context.requestTime"
      httpMethod     = "$context.httpMethod"
      resourcePath   = "$context.resourcePath"
      status         = "$context.status"
      protocol       = "$context.protocol"
      responseLength = "$context.responseLength"
      integrationLatency = "$context.integrationLatency"
      responseLatency = "$context.responseLatency"
    })
  }

  # Stage variables
  variables = {
    lambdaAlias     = var.environment
    stageName       = var.environment
    tableNameSuffix = var.environment
  }

  # Enable X-Ray tracing
  xray_tracing_enabled = true

  # Tags
  tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}
```

### Stage-specific Method Settings

```hcl
resource "aws_api_gateway_method_settings" "example" {
  rest_api_id = aws_api_gateway_rest_api.example_api.id
  stage_name  = aws_api_gateway_stage.example.stage_name
  method_path = "*/*"  # Apply to all methods

  settings {
    # Logging settings
    metrics_enabled        = true
    logging_level          = "INFO"  # OFF, ERROR, INFO
    data_trace_enabled     = true

    # Throttling settings
    throttling_burst_limit = 5000
    throttling_rate_limit  = 10000

    # Caching settings
    caching_enabled        = true
    cache_ttl_in_seconds   = 300

    # Detailed metrics
    metrics_enabled        = true
  }
}
```

## Monitoring and Logging

### CloudWatch Logs

```hcl
resource "aws_cloudwatch_log_group" "api_gateway_logs" {
  name              = "/aws/apigateway/${var.project_name}-${var.environment}"
  retention_in_days = 30

  tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}
```

### CloudWatch Alarms

```hcl
# Alarm for 4xx errors
resource "aws_cloudwatch_metric_alarm" "api_4xx_errors" {
  alarm_name          = "${var.project_name}-${var.environment}-4xx-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "4XXError"
  namespace           = "AWS/ApiGateway"
  period              = 60
  statistic           = "Sum"
  threshold           = 10
  alarm_description   = "This metric monitors API Gateway 4XX errors"

  dimensions = {
    ApiName  = aws_api_gateway_rest_api.example_api.name
    Stage    = aws_api_gateway_stage.example.stage_name
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}

# Alarm for 5xx errors
resource "aws_cloudwatch_metric_alarm" "api_5xx_errors" {
  alarm_name          = "${var.project_name}-${var.environment}-5xx-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "5XXError"
  namespace           = "AWS/ApiGateway"
  period              = 60
  statistic           = "Sum"
  threshold           = 5
  alarm_description   = "This metric monitors API Gateway 5XX errors"

  dimensions = {
    ApiName  = aws_api_gateway_rest_api.example_api.name
    Stage    = aws_api_gateway_stage.example.stage_name
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}

# Alarm for latency
resource "aws_cloudwatch_metric_alarm" "api_latency" {
  alarm_name          = "${var.project_name}-${var.environment}-latency"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "Latency"
  namespace           = "AWS/ApiGateway"
  period              = 60
  statistic           = "Average"
  threshold           = 1000  # 1 second
  alarm_description   = "This metric monitors API Gateway latency"

  dimensions = {
    ApiName  = aws_api_gateway_rest_api.example_api.name
    Stage    = aws_api_gateway_stage.example.stage_name
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

## Performance Optimization

### Caching

```hcl
# Enable caching for specific methods
resource "aws_api_gateway_method_settings" "caching_example" {
  rest_api_id = aws_api_gateway_rest_api.example_api.id
  stage_name  = aws_api_gateway_stage.example.stage_name
  method_path = "items/GET"  # Specific method path

  settings {
    caching_enabled      = true
    cache_ttl_in_seconds = 300
    cache_data_encrypted = true
  }
}
```

### Throttling

```hcl
# Set throttling limits for specific methods
resource "aws_api_gateway_method_settings" "throttling_example" {
  rest_api_id = aws_api_gateway_rest_api.example_api.id
  stage_name  = aws_api_gateway_stage.example.stage_name
  method_path = "items/POST"  # Specific method path

  settings {
    throttling_burst_limit = 100
    throttling_rate_limit  = 50
  }
}
```

## Security Best Practices

### WAF Integration

```hcl
resource "aws_wafv2_web_acl_association" "api_gateway" {
  resource_arn = aws_api_gateway_stage.example.arn
  web_acl_arn  = aws_wafv2_web_acl.api_waf.arn
}
```

### Resource Policy

```hcl
resource "aws_api_gateway_rest_api_policy" "example" {
  rest_api_id = aws_api_gateway_rest_api.example_api.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect    = "Allow"
        Principal = "*"
        Action    = "execute-api:Invoke"
        Resource  = "${aws_api_gateway_rest_api.example_api.execution_arn}/*"
        Condition = {
          IpAddress = {
            "aws:SourceIp" = var.allowed_ip_ranges
          }
        }
      }
    ]
  })
}
```

### Client Certificate

```hcl
resource "aws_api_gateway_client_certificate" "example" {
  description = "Client certificate for ${var.project_name}-${var.environment}"
}

# Associate with stage
resource "aws_api_gateway_stage" "with_client_cert" {
  # ... other configuration ...
  client_certificate_id = aws_api_gateway_client_certificate.example.id
}
```

## Cost Optimization

1. **Use Caching**: Enable caching for frequently accessed endpoints
2. **Optimize Lambda Integration**: Minimize Lambda execution time
3. **Monitor Usage**: Set up CloudWatch dashboards to track API usage
4. **Use Usage Plans**: Implement throttling to prevent unexpected costs
5. **Clean Up Unused Resources**: Remove unused APIs, stages, and deployments

## Common Patterns and Examples

### REST API with Multiple Resources

```hcl
# Define a module for creating API resources and methods
module "api_resource" {
  source = "./modules/api_resource"

  for_each = {
    "users"    = { parent_id = aws_api_gateway_rest_api.example_api.root_resource_id, methods = ["GET", "POST"] }
    "products" = { parent_id = aws_api_gateway_rest_api.example_api.root_resource_id, methods = ["GET", "POST", "PUT"] }
    "orders"   = { parent_id = aws_api_gateway_rest_api.example_api.root_resource_id, methods = ["GET", "POST"] }
  }

  rest_api_id = aws_api_gateway_rest_api.example_api.id
  parent_id   = each.value.parent_id
  path_part   = each.key
  methods     = each.value.methods
  lambda_function_arns = {
    "GET"  = aws_lambda_function.get_handler.invoke_arn
    "POST" = aws_lambda_function.post_handler.invoke_arn
    "PUT"  = aws_lambda_function.put_handler.invoke_arn
  }
}
```

### API Documentation

```hcl
# API documentation
resource "aws_api_gateway_documentation_part" "example" {
  rest_api_id = aws_api_gateway_rest_api.example_api.id
  location {
    type   = "METHOD"
    method = "GET"
    path   = "/items"
  }

  properties = jsonencode({
    description = "Get a list of items"
    summary     = "List Items"
    operationId = "listItems"
    parameters  = [
      {
        name        = "limit"
        in          = "query"
        description = "Maximum number of items to return"
        required    = false
        schema      = { type = "integer" }
      }
    ]
  })
}

# Documentation version
resource "aws_api_gateway_documentation_version" "example" {
  rest_api_id = aws_api_gateway_rest_api.example_api.id
  version     = "1.0.0"
  description = "Initial version"

  depends_on = [
    aws_api_gateway_documentation_part.example
  ]
}
```

### Import OpenAPI/Swagger Definition

```hcl
resource "aws_api_gateway_rest_api" "from_swagger" {
  name        = "${var.project_name}-${var.environment}-api"
  description = "API created from Swagger/OpenAPI definition"

  body = templatefile("${path.module}/swagger.json", {
    lambda_uri = aws_lambda_function.example.invoke_arn
    region     = var.aws_region
    account_id = data.aws_caller_identity.current.account_id
  })

  endpoint_configuration {
    types = ["REGIONAL"]
  }
}
```

---

For more information on integrating API Gateway with other AWS services, refer to the other documents in this repository:
- [Lambda Integration](lambda.md)
- [IAM Best Practices](iam.md)
- [Monitoring and Observability](monitoring.md)
