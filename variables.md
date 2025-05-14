---
title: "Variables Management Best Practices for Terraform AWS Projects"
category: "Terraform"
tags: ["aws", "terraform", "variables", "best practices", "infrastructure as code"]
last_updated: "2025-05-14"
---

# Variables Management Best Practices for Terraform AWS Projects

This document outlines the best practices for managing variables in Terraform projects, particularly for AWS API Gateway and Lambda deployments. Following these guidelines will help ensure your Terraform configurations are maintainable, reusable, and properly organized.

## Table of Contents

- [Variable Organization](#variable-organization)
- [Variable Types and Validation](#variable-types-and-validation)
- [Environment-Specific Variables](#environment-specific-variables)
- [Sensitive Data Handling](#sensitive-data-handling)
- [Variable Defaults and Required Variables](#variable-defaults-and-required-variables)
- [Local Variables](#local-variables)
- [Variable Naming Conventions](#variable-naming-conventions)
- [Module Variables](#module-variables)
- [Variable Documentation](#variable-documentation)
- [Common Variable Patterns](#common-variable-patterns)

## Variable Organization

### Project Structure

Organize your variables into logical files:

```
project/
├── main.tf
├── variables.tf        # Common variables
├── outputs.tf
├── environments/
│   ├── dev.tfvars      # Development environment variables
│   ├── staging.tfvars  # Staging environment variables
│   └── prod.tfvars     # Production environment variables
├── modules/
│   ├── api_gateway/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── lambda/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── terraform.tfvars    # Default values (should not contain sensitive data)
```

### Variables File Structure

```hcl
# variables.tf

# Project metadata
variable "project_name" {
  description = "Name of the project"
  type        = string
}

variable "environment" {
  description = "Deployment environment (dev, staging, prod)"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}

# AWS configuration
variable "aws_region" {
  description = "AWS region to deploy resources"
  type        = string
  default     = "us-east-1"
}

# Resource-specific variables
variable "lambda_config" {
  description = "Configuration for Lambda functions"
  type        = map(object({
    runtime     = string
    memory_size = number
    timeout     = number
  }))
}

variable "api_gateway_config" {
  description = "Configuration for API Gateway"
  type        = object({
    name        = string
    description = string
    endpoint_type = string
  })
}
```

## Variable Types and Validation

### Basic Types

```hcl
# String variable
variable "function_name" {
  description = "Name of the Lambda function"
  type        = string
}

# Number variable
variable "memory_size" {
  description = "Memory size for Lambda function in MB"
  type        = number
  default     = 128
}

# Boolean variable
variable "enable_xray" {
  description = "Enable X-Ray tracing"
  type        = bool
  default     = false
}

# List variable
variable "subnet_ids" {
  description = "List of subnet IDs for Lambda VPC configuration"
  type        = list(string)
  default     = []
}

# Map variable
variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

### Complex Types

```hcl
# Object variable
variable "lambda_function" {
  description = "Lambda function configuration"
  type = object({
    name        = string
    runtime     = string
    memory_size = number
    timeout     = number
    environment_variables = map(string)
  })
}

# List of objects
variable "api_routes" {
  description = "API Gateway routes configuration"
  type = list(object({
    path        = string
    http_method = string
    lambda_name = string
    auth_type   = string
  }))
}

# Map of objects
variable "lambda_functions" {
  description = "Map of Lambda function configurations"
  type = map(object({
    runtime     = string
    memory_size = number
    timeout     = number
    handler     = string
  }))
}

# Tuple type
variable "cors_configuration" {
  description = "CORS configuration as [allowed_origins, allowed_methods, allowed_headers]"
  type        = tuple([list(string), list(string), list(string)])
  default     = [["*"], ["GET", "POST", "OPTIONS"], ["Content-Type", "Authorization"]]
}
```

### Variable Validation

```hcl
# Validate memory size is within AWS limits
variable "memory_size" {
  description = "Memory size for Lambda function in MB"
  type        = number
  default     = 128

  validation {
    condition     = var.memory_size >= 128 && var.memory_size <= 10240 && var.memory_size % 64 == 0
    error_message = "Memory size must be between 128 and 10240 MB and a multiple of 64 MB."
  }
}

# Validate AWS region
variable "aws_region" {
  description = "AWS region to deploy resources"
  type        = string
  default     = "us-east-1"

  validation {
    condition     = can(regex("^(us|eu|ap|sa|ca|me|af)-[a-z]+-[0-9]+$", var.aws_region))
    error_message = "Must be a valid AWS region name."
  }
}

# Validate Lambda runtime
variable "runtime" {
  description = "Lambda function runtime"
  type        = string

  validation {
    condition     = contains(["nodejs18.x", "nodejs16.x", "python3.9", "python3.8", "java11", "dotnet6"], var.runtime)
    error_message = "Runtime must be one of the supported Lambda runtimes."
  }
}
```

## Environment-Specific Variables

### Environment Variable Files

Create separate `.tfvars` files for each environment:

```hcl
# environments/dev.tfvars
environment = "dev"
aws_region  = "us-east-1"

lambda_config = {
  api_handler = {
    runtime     = "nodejs18.x"
    memory_size = 128
    timeout     = 30
  }
}

api_gateway_config = {
  name          = "dev-api"
  description   = "Development API"
  endpoint_type = "REGIONAL"
}
```

```hcl
# environments/prod.tfvars
environment = "prod"
aws_region  = "us-east-1"

lambda_config = {
  api_handler = {
    runtime     = "nodejs18.x"
    memory_size = 512
    timeout     = 60
  }
}

api_gateway_config = {
  name          = "prod-api"
  description   = "Production API"
  endpoint_type = "REGIONAL"
}
```

### Using Environment Variables

Apply environment-specific variables:

```bash
terraform apply -var-file=environments/dev.tfvars
```

### Workspace-Based Configuration

```hcl
# variables.tf
variable "environment_config" {
  description = "Environment-specific configuration"
  type = map(object({
    lambda_memory = number
    lambda_timeout = number
    api_throttling_rate = number
  }))

  default = {
    dev = {
      lambda_memory = 128
      lambda_timeout = 30
      api_throttling_rate = 100
    }
    staging = {
      lambda_memory = 256
      lambda_timeout = 60
      api_throttling_rate = 500
    }
    prod = {
      lambda_memory = 512
      lambda_timeout = 60
      api_throttling_rate = 1000
    }
  }
}

# main.tf
locals {
  # Use workspace name to select configuration
  env = terraform.workspace
  config = var.environment_config[local.env]
}

resource "aws_lambda_function" "example" {
  # ... other configuration ...
  memory_size = local.config.lambda_memory
  timeout     = local.config.lambda_timeout
}
```

## Sensitive Data Handling

### Marking Variables as Sensitive

```hcl
variable "database_password" {
  description = "Password for the database"
  type        = string
  sensitive   = true
}

variable "api_keys" {
  description = "API keys for external services"
  type        = map(string)
  sensitive   = true
}
```

### Using Environment Variables for Sensitive Data

```bash
# Set sensitive values as environment variables
export TF_VAR_database_password="supersecretpassword"
export TF_VAR_api_keys='{"service1":"key1","service2":"key2"}'

# Run Terraform without exposing sensitive data in command line
terraform apply -var-file=environments/dev.tfvars
```

### Using AWS Secrets Manager or Parameter Store

```hcl
# Retrieve secrets from AWS Secrets Manager
data "aws_secretsmanager_secret" "db_credentials" {
  name = "${var.project_name}/${var.environment}/db-credentials"
}

data "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = data.aws_secretsmanager_secret.db_credentials.id
}

locals {
  db_credentials = jsondecode(data.aws_secretsmanager_secret_version.db_credentials.secret_string)
}

# Use the secrets in resources
resource "aws_lambda_function" "example" {
  # ... other configuration ...

  environment {
    variables = {
      DB_HOST     = local.db_credentials.host
      DB_USERNAME = local.db_credentials.username
      DB_PASSWORD = local.db_credentials.password
    }
  }
}
```

## Variable Defaults and Required Variables

### Default Values

```hcl
# Variable with default value
variable "lambda_timeout" {
  description = "Timeout for Lambda functions in seconds"
  type        = number
  default     = 30
}

# Variable with conditional default
variable "log_retention_days" {
  description = "Number of days to retain logs"
  type        = number
  default     = var.environment == "prod" ? 90 : 30
}
```

### Required Variables

```hcl
# Required variable (no default)
variable "project_name" {
  description = "Name of the project"
  type        = string

  validation {
    condition     = length(var.project_name) > 0
    error_message = "Project name cannot be empty."
  }
}

# Required variable with validation
variable "environment" {
  description = "Deployment environment"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}
```

## Local Variables

### Basic Local Variables

```hcl
locals {
  # Naming convention for resources
  name_prefix = "${var.project_name}-${var.environment}"

  # Common tags for all resources
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
    Owner       = var.owner
  }

  # Merge common tags with resource-specific tags
  lambda_tags = merge(local.common_tags, {
    Service = "lambda"
  })

  # Computed values
  is_production = var.environment == "prod"
  log_retention = local.is_production ? 90 : 30
}
```

### Complex Transformations

```hcl
locals {
  # Transform list to map
  lambda_functions_map = {
    for func in var.lambda_functions :
    func.name => func
  }

  # Filter list based on condition
  production_lambdas = [
    for func in var.lambda_functions :
    func if func.environment == "prod"
  ]

  # Create flattened list from nested structure
  all_subnet_ids = flatten([
    for vpc in var.vpc_config :
    vpc.subnet_ids
  ])

  # Create map with computed values
  api_routes_with_arn = {
    for route_key, route in var.api_routes :
    route_key => {
      path        = route.path
      http_method = route.http_method
      lambda_arn  = aws_lambda_function.functions[route.lambda_name].arn
    }
  }
}
```

## Variable Naming Conventions

### General Naming Guidelines

- Use snake_case for variable names
- Use descriptive names that indicate purpose
- Group related variables with common prefixes
- Use singular nouns for single values, plural for collections

### Examples

```hcl
# Good variable names
variable "project_name" {}
variable "environment" {}
variable "lambda_memory_size" {}
variable "api_gateway_endpoint_type" {}
variable "subnet_ids" {}
variable "enable_xray_tracing" {}

# Avoid these patterns
variable "pname" {}                # Too short/unclear
variable "lambda_mem" {}           # Abbreviated
variable "ApiGatewayEndpointType" {}  # Not snake_case
variable "the_subnet_ids_for_vpc" {}  # Too verbose
```

## Module Variables

### Input Variables for Modules

```hcl
# modules/lambda/variables.tf
variable "function_name" {
  description = "Name of the Lambda function"
  type        = string
}

variable "runtime" {
  description = "Lambda runtime"
  type        = string
  default     = "nodejs18.x"
}

variable "memory_size" {
  description = "Memory size in MB"
  type        = number
  default     = 128
}

variable "timeout" {
  description = "Function timeout in seconds"
  type        = number
  default     = 30
}

variable "environment_variables" {
  description = "Environment variables for the Lambda function"
  type        = map(string)
  default     = {}
}

variable "tags" {
  description = "Tags to apply to the Lambda function"
  type        = map(string)
  default     = {}
}
```

### Module Usage

```hcl
# main.tf
module "api_lambda" {
  source = "./modules/lambda"

  function_name         = "${local.name_prefix}-api-handler"
  runtime               = "nodejs18.x"
  memory_size           = var.environment == "prod" ? 512 : 128
  timeout               = 60
  environment_variables = {
    ENVIRONMENT = var.environment
    LOG_LEVEL   = var.environment == "prod" ? "INFO" : "DEBUG"
    API_URL     = module.api_gateway.api_url
  }
  tags                  = local.common_tags
}
```

## Variable Documentation

### Comprehensive Documentation

```hcl
variable "lambda_config" {
  description = <<-EOT
    Configuration for Lambda functions.

    The map key is the function name suffix, and the value is an object with the following properties:
    - runtime: The Lambda runtime to use (e.g., nodejs18.x, python3.9)
    - memory_size: Memory allocation in MB (128-10240)
    - timeout: Function timeout in seconds (1-900)
    - environment_variables: Map of environment variables for the function

    Example:
    ```
    lambda_config = {
      api_handler = {
        runtime     = "nodejs18.x"
        memory_size = 512
        timeout     = 60
        environment_variables = {
          LOG_LEVEL = "INFO"
        }
      }
    }
    ```
  EOT

  type = map(object({
    runtime              = string
    memory_size          = number
    timeout              = number
    environment_variables = optional(map(string), {})
  }))

  # Simple validation example - in practice, use more specific validations
  validation {
    condition     = length(var.lambda_config) > 0
    error_message = "At least one Lambda function configuration must be provided."
  }
}
```

## Common Variable Patterns

### Feature Flags

```hcl
variable "features" {
  description = "Feature flags for enabling/disabling functionality"
  type = object({
    enable_xray_tracing = bool
    enable_vpc_access   = bool
    enable_api_caching  = bool
    enable_waf          = bool
  })

  default = {
    enable_xray_tracing = false
    enable_vpc_access   = false
    enable_api_caching  = false
    enable_waf          = false
  }
}

# Usage
resource "aws_lambda_function" "example" {
  # ... other configuration ...

  dynamic "tracing_config" {
    for_each = var.features.enable_xray_tracing ? [1] : []
    content {
      mode = "Active"
    }
  }

  dynamic "vpc_config" {
    for_each = var.features.enable_vpc_access ? [1] : []
    content {
      subnet_ids         = var.subnet_ids
      security_group_ids = [aws_security_group.lambda.id]
    }
  }
}
```

### Resource Sizing by Environment

```hcl
variable "sizing" {
  description = "Resource sizing by environment"
  type = map(object({
    lambda_memory   = number
    lambda_timeout  = number
    api_burst_limit = number
    api_rate_limit  = number
  }))

  default = {
    dev = {
      lambda_memory   = 128
      lambda_timeout  = 30
      api_burst_limit = 10
      api_rate_limit  = 5
    }
    staging = {
      lambda_memory   = 256
      lambda_timeout  = 60
      api_burst_limit = 20
      api_rate_limit  = 10
    }
    prod = {
      lambda_memory   = 512
      lambda_timeout  = 60
      api_burst_limit = 100
      api_rate_limit  = 50
    }
  }
}

# Usage
locals {
  size_config = var.sizing[var.environment]
}

resource "aws_lambda_function" "example" {
  # ... other configuration ...
  memory_size = local.size_config.lambda_memory
  timeout     = local.size_config.lambda_timeout
}

resource "aws_api_gateway_method_settings" "example" {
  # ... other configuration ...
  settings {
    throttling_burst_limit = local.size_config.api_burst_limit
    throttling_rate_limit  = local.size_config.api_rate_limit
  }
}
```

### Regional Configuration

```hcl
variable "regional_config" {
  description = "Region-specific configuration"
  type = map(object({
    subnet_ids         = list(string)
    security_group_ids = list(string)
    zone_id            = string
    certificate_arn    = string
  }))
}

# Usage
locals {
  region_config = var.regional_config[var.aws_region]
}

resource "aws_lambda_function" "example" {
  # ... other configuration ...

  vpc_config {
    subnet_ids         = local.region_config.subnet_ids
    security_group_ids = local.region_config.security_group_ids
  }
}

resource "aws_api_gateway_domain_name" "example" {
  # ... other configuration ...
  regional_certificate_arn = local.region_config.certificate_arn
}
```

---

For more information on integrating with other AWS services, refer to the other documents in this repository:
- [API Gateway Guidelines](api_gateway.md)
- [Lambda Guidelines](lambda.md)
- [Provider Configuration](provider.md)
- [IAM Best Practices](iam.md)
