# Terraform Outputs Best Practices for AWS API Gateway and Lambda

This document outlines the best practices for defining and managing outputs in Terraform projects, particularly for AWS API Gateway and Lambda deployments. Following these guidelines will help ensure your Terraform configurations are well-documented, maintainable, and provide useful information to users of your modules.

## Table of Contents

- [Output Organization](#output-organization)
- [Output Naming Conventions](#output-naming-conventions)
- [Essential Outputs](#essential-outputs)
- [Sensitive Outputs](#sensitive-outputs)
- [Output Description and Documentation](#output-description-and-documentation)
- [Module Outputs](#module-outputs)
- [Output Dependencies](#output-dependencies)
- [Output Formatting](#output-formatting)
- [Common Output Patterns](#common-output-patterns)

## Output Organization

### Project Structure

Organize your outputs into logical files:

```
project/
├── main.tf
├── variables.tf
├── outputs.tf        # Main outputs file
├── modules/
│   ├── api_gateway/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf  # Module-specific outputs
│   └── lambda/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf  # Module-specific outputs
```

### Outputs File Structure

Group related outputs together and organize them logically:

```hcl
# outputs.tf

# API Gateway outputs
output "api_gateway_id" {
  description = "ID of the API Gateway REST API"
  value       = aws_api_gateway_rest_api.main.id
}

output "api_gateway_endpoint" {
  description = "Endpoint URL of the API Gateway"
  value       = aws_api_gateway_deployment.main.invoke_url
}

# Lambda outputs
output "lambda_function_names" {
  description = "Names of the Lambda functions"
  value       = [for func in aws_lambda_function.functions : func.function_name]
}

output "lambda_function_arns" {
  description = "ARNs of the Lambda functions"
  value       = [for func in aws_lambda_function.functions : func.arn]
}

# General outputs
output "deployment_environment" {
  description = "Environment where resources are deployed"
  value       = var.environment
}
```

## Output Naming Conventions

### General Naming Guidelines

- Use snake_case for output names
- Use descriptive names that indicate purpose
- Group related outputs with common prefixes
- Use singular nouns for single values, plural for collections

### Examples

```hcl
# Good output names
output "api_gateway_id" {}
output "api_gateway_endpoint" {}
output "lambda_function_name" {}
output "lambda_function_arns" {}  # Plural for a list
output "cloudwatch_log_group_names" {}

# Avoid these patterns
output "api_id" {}                # Too short/unclear
output "lambda_func_arn" {}       # Abbreviated
output "ApiGatewayEndpoint" {}    # Not snake_case
output "the_lambda_function_name" {}  # Too verbose
```

## Essential Outputs

### API Gateway Essential Outputs

```hcl
output "api_gateway_id" {
  description = "ID of the API Gateway REST API"
  value       = aws_api_gateway_rest_api.main.id
}

output "api_gateway_root_resource_id" {
  description = "ID of the API Gateway root resource"
  value       = aws_api_gateway_rest_api.main.root_resource_id
}

output "api_gateway_execution_arn" {
  description = "Execution ARN of the API Gateway"
  value       = aws_api_gateway_rest_api.main.execution_arn
}

output "api_gateway_endpoint" {
  description = "Endpoint URL of the API Gateway"
  value       = "${aws_api_gateway_deployment.main.invoke_url}${aws_api_gateway_stage.main.stage_name}"
}

output "api_gateway_stage_name" {
  description = "Name of the API Gateway stage"
  value       = aws_api_gateway_stage.main.stage_name
}

output "api_gateway_custom_domain" {
  description = "Custom domain name for the API Gateway (if configured)"
  value       = try(aws_api_gateway_domain_name.custom[0].domain_name, null)
}
```

### Lambda Essential Outputs

```hcl
output "lambda_function_name" {
  description = "Name of the Lambda function"
  value       = aws_lambda_function.main.function_name
}

output "lambda_function_arn" {
  description = "ARN of the Lambda function"
  value       = aws_lambda_function.main.arn
}

output "lambda_function_invoke_arn" {
  description = "Invoke ARN of the Lambda function"
  value       = aws_lambda_function.main.invoke_arn
}

output "lambda_function_qualified_arn" {
  description = "Qualified ARN of the Lambda function including version"
  value       = aws_lambda_function.main.qualified_arn
}

output "lambda_function_version" {
  description = "Latest published version of the Lambda function"
  value       = aws_lambda_function.main.version
}

output "lambda_function_alias_name" {
  description = "Name of the Lambda function alias"
  value       = try(aws_lambda_alias.main[0].name, null)
}

output "lambda_function_role_arn" {
  description = "ARN of the IAM role attached to the Lambda function"
  value       = aws_lambda_function.main.role
}

output "lambda_function_log_group_name" {
  description = "Name of the CloudWatch log group for the Lambda function"
  value       = "/aws/lambda/${aws_lambda_function.main.function_name}"
}
```

### IAM Role Outputs

```hcl
output "lambda_execution_role_id" {
  description = "ID of the Lambda execution IAM role"
  value       = aws_iam_role.lambda_execution.id
}

output "lambda_execution_role_arn" {
  description = "ARN of the Lambda execution IAM role"
  value       = aws_iam_role.lambda_execution.arn
}

output "api_gateway_role_id" {
  description = "ID of the API Gateway IAM role"
  value       = aws_iam_role.api_gateway.id
}

output "api_gateway_role_arn" {
  description = "ARN of the API Gateway IAM role"
  value       = aws_iam_role.api_gateway.arn
}
```

## Sensitive Outputs

### Marking Outputs as Sensitive

```hcl
output "database_connection_string" {
  description = "Connection string for the database"
  value       = "postgresql://${var.db_username}:${var.db_password}@${aws_db_instance.main.endpoint}/${aws_db_instance.main.name}"
  sensitive   = true
}

output "api_key" {
  description = "API key for the API Gateway"
  value       = aws_api_gateway_api_key.main.value
  sensitive   = true
}
```

### Handling Sensitive Outputs

```hcl
# Store sensitive outputs in AWS Secrets Manager
resource "aws_secretsmanager_secret" "api_credentials" {
  name = "${var.project_name}/${var.environment}/api-credentials"
}

resource "aws_secretsmanager_secret_version" "api_credentials" {
  secret_id     = aws_secretsmanager_secret.api_credentials.id
  secret_string = jsonencode({
    api_key = aws_api_gateway_api_key.main.value
    api_endpoint = aws_api_gateway_deployment.main.invoke_url
  })
}

# Output the secret ARN instead of the actual values
output "api_credentials_secret_arn" {
  description = "ARN of the secret containing API credentials"
  value       = aws_secretsmanager_secret.api_credentials.arn
}
```

## Output Description and Documentation

### Comprehensive Descriptions

```hcl
output "api_gateway_endpoint" {
  description = <<-EOT
    The invoke URL for the API Gateway deployment. This URL can be used to access the API.
    Format: https://{api-id}.execute-api.{region}.amazonaws.com/{stage-name}
    
    Example usage:
    ```
    curl -X GET "${api_gateway_endpoint}/items"
    ```
  EOT
  value       = "${aws_api_gateway_deployment.main.invoke_url}${aws_api_gateway_stage.main.stage_name}"
}

output "lambda_function_details" {
  description = <<-EOT
    A map containing details about the Lambda function:
    - name: The function name
    - arn: The function ARN
    - version: The function version
    - last_modified: When the function was last modified
    - runtime: The runtime environment
    - memory_size: Allocated memory in MB
    - timeout: Function timeout in seconds
  EOT
  value = {
    name          = aws_lambda_function.main.function_name
    arn           = aws_lambda_function.main.arn
    version       = aws_lambda_function.main.version
    last_modified = aws_lambda_function.main.last_modified
    runtime       = aws_lambda_function.main.runtime
    memory_size   = aws_lambda_function.main.memory_size
    timeout       = aws_lambda_function.main.timeout
  }
}
```

## Module Outputs

### Module Output Structure

```hcl
# modules/lambda/outputs.tf
output "function_name" {
  description = "Name of the Lambda function"
  value       = aws_lambda_function.main.function_name
}

output "function_arn" {
  description = "ARN of the Lambda function"
  value       = aws_lambda_function.main.arn
}

output "invoke_arn" {
  description = "Invoke ARN of the Lambda function"
  value       = aws_lambda_function.main.invoke_arn
}

output "execution_role_arn" {
  description = "ARN of the Lambda execution role"
  value       = aws_iam_role.lambda_execution.arn
}

output "log_group_name" {
  description = "Name of the CloudWatch log group for the Lambda function"
  value       = aws_cloudwatch_log_group.lambda.name
}
```

### Using Module Outputs

```hcl
# main.tf
module "api_lambda" {
  source = "./modules/lambda"
  # ... input variables ...
}

# Reference module outputs
output "api_lambda_function_name" {
  description = "Name of the API Lambda function"
  value       = module.api_lambda.function_name
}

output "api_lambda_invoke_arn" {
  description = "Invoke ARN of the API Lambda function"
  value       = module.api_lambda.invoke_arn
}

# Use module outputs in other resources
resource "aws_api_gateway_integration" "lambda_integration" {
  # ... other configuration ...
  uri                     = module.api_lambda.invoke_arn
}
```

## Output Dependencies

### Explicit Dependencies

```hcl
output "api_gateway_endpoint" {
  description = "Endpoint URL of the API Gateway"
  value       = "${aws_api_gateway_deployment.main.invoke_url}${aws_api_gateway_stage.main.stage_name}"
  
  # Ensure the output is only available after the deployment is complete
  depends_on = [
    aws_api_gateway_deployment.main,
    aws_api_gateway_stage.main
  ]
}
```

### Conditional Outputs

```hcl
output "custom_domain_endpoint" {
  description = "Custom domain endpoint for the API Gateway"
  value       = var.create_custom_domain ? "https://${aws_api_gateway_domain_name.custom[0].domain_name}" : null
}

output "lambda_layer_arn" {
  description = "ARN of the Lambda layer (if created)"
  value       = var.create_lambda_layer ? aws_lambda_layer_version.layer[0].arn : null
}
```

## Output Formatting

### Structured Outputs

```hcl
output "lambda_functions" {
  description = "Map of Lambda functions with their details"
  value = {
    for name, func in aws_lambda_function.functions : name => {
      arn         = func.arn
      invoke_arn  = func.invoke_arn
      version     = func.version
      runtime     = func.runtime
      memory_size = func.memory_size
      timeout     = func.timeout
    }
  }
}

output "api_gateway_resources" {
  description = "Map of API Gateway resources with their details"
  value = {
    for path, resource in aws_api_gateway_resource.resources : path => {
      id        = resource.id
      path      = resource.path
      path_part = resource.path_part
      parent_id = resource.parent_id
    }
  }
}
```

### Formatted Strings

```hcl
output "api_gateway_url" {
  description = "URL for the API Gateway"
  value       = format("https://%s.execute-api.%s.amazonaws.com/%s", 
                       aws_api_gateway_rest_api.main.id,
                       var.aws_region,
                       aws_api_gateway_stage.main.stage_name)
}

output "lambda_log_group_arn" {
  description = "ARN of the Lambda CloudWatch log group"
  value       = format("arn:aws:logs:%s:%s:log-group:/aws/lambda/%s:*",
                       var.aws_region,
                       data.aws_caller_identity.current.account_id,
                       aws_lambda_function.main.function_name)
}
```

## Common Output Patterns

### Environment-Specific Outputs

```hcl
output "environment_details" {
  description = "Details about the deployment environment"
  value = {
    name        = var.environment
    region      = var.aws_region
    account_id  = data.aws_caller_identity.current.account_id
    is_prod     = var.environment == "prod"
    api_url     = "${aws_api_gateway_deployment.main.invoke_url}${aws_api_gateway_stage.main.stage_name}"
    log_level   = var.environment == "prod" ? "INFO" : "DEBUG"
  }
}
```

### Resource Mapping Outputs

```hcl
output "lambda_api_mappings" {
  description = "Mapping of API paths to Lambda functions"
  value = {
    for path, integration in aws_api_gateway_integration.lambda : path => {
      http_method   = integration.http_method
      resource_path = aws_api_gateway_resource.resources[path].path
      lambda_arn    = aws_lambda_function.functions[path].arn
    }
  }
}
```

### Deployment Information

```hcl
output "deployment_info" {
  description = "Information about the deployment"
  value = {
    timestamp   = timestamp()
    environment = var.environment
    region      = var.aws_region
    terraform_version = terraform.required_version
    api_gateway = {
      id        = aws_api_gateway_rest_api.main.id
      stage     = aws_api_gateway_stage.main.stage_name
      endpoint  = "${aws_api_gateway_deployment.main.invoke_url}${aws_api_gateway_stage.main.stage_name}"
    }
    lambda_functions = [
      for func in aws_lambda_function.functions : {
        name    = func.function_name
        runtime = func.runtime
        version = func.version
      }
    ]
  }
}
```

### Cross-Stack References

```hcl
output "terraform_remote_state_values" {
  description = "Values to be used in terraform_remote_state in other stacks"
  value = {
    api_gateway_id         = aws_api_gateway_rest_api.main.id
    api_gateway_stage_name = aws_api_gateway_stage.main.stage_name
    lambda_function_arns   = {
      for name, func in aws_lambda_function.functions : name => func.arn
    }
    vpc_id                 = var.vpc_id
    subnet_ids             = var.subnet_ids
    security_group_ids     = [aws_security_group.lambda.id]
  }
}
```

### CLI-Friendly Outputs

```hcl
output "curl_commands" {
  description = "Example curl commands to test the API"
  value = {
    get_items    = "curl -X GET '${aws_api_gateway_deployment.main.invoke_url}${aws_api_gateway_stage.main.stage_name}/items'"
    get_item     = "curl -X GET '${aws_api_gateway_deployment.main.invoke_url}${aws_api_gateway_stage.main.stage_name}/items/{id}'"
    create_item  = "curl -X POST '${aws_api_gateway_deployment.main.invoke_url}${aws_api_gateway_stage.main.stage_name}/items' -H 'Content-Type: application/json' -d '{\"name\":\"test\",\"description\":\"test item\"}'"
    update_item  = "curl -X PUT '${aws_api_gateway_deployment.main.invoke_url}${aws_api_gateway_stage.main.stage_name}/items/{id}' -H 'Content-Type: application/json' -d '{\"name\":\"updated\",\"description\":\"updated item\"}'"
    delete_item  = "curl -X DELETE '${aws_api_gateway_deployment.main.invoke_url}${aws_api_gateway_stage.main.stage_name}/items/{id}'"
  }
}

output "aws_cli_commands" {
  description = "Example AWS CLI commands to interact with the resources"
  value = {
    invoke_lambda = "aws lambda invoke --function-name ${aws_lambda_function.main.function_name} --payload '{\"key\":\"value\"}' response.json"
    get_logs      = "aws logs get-log-events --log-group-name /aws/lambda/${aws_lambda_function.main.function_name} --log-stream-name <log-stream-name>"
    update_lambda = "aws lambda update-function-code --function-name ${aws_lambda_function.main.function_name} --zip-file fileb://function.zip"
  }
}
```

---

For more information on integrating with other AWS services, refer to the other documents in this repository:
- [API Gateway Guidelines](api_gateway.md)
- [Lambda Guidelines](lambda.md)
- [Variables Management](variables.md)
- [IAM Best Practices](iam.md)
