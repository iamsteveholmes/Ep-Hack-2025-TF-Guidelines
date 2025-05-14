---
title: "AWS Provider Configuration Best Practices for Terraform"
category: "Terraform"
tags: ["aws", "terraform", "provider", "state management", "security", "best practices"]
last_updated: "2025-05-14"
---

# AWS Provider Configuration Best Practices for Terraform

This document outlines the best practices for configuring the AWS provider in Terraform projects, particularly for API Gateway and Lambda deployments. Following these guidelines will help ensure your Terraform configurations are secure, maintainable, and follow infrastructure as code best practices.

## Table of Contents

- [Provider Configuration](#provider-configuration)
- [Authentication Methods](#authentication-methods)
- [State Management](#state-management)
- [Provider Versioning](#provider-versioning)
- [Multi-Region Deployments](#multi-region-deployments)
- [Multi-Account Deployments](#multi-account-deployments)
- [Workspace Management](#workspace-management)
- [Security Best Practices](#security-best-practices)

## Provider Configuration

### Basic Provider Configuration

```hcl
provider "aws" {
  region = var.aws_region

  # Optional: Specify the AWS profile to use
  profile = var.aws_profile

  # Default tags applied to all resources
  default_tags {
    tags = {
      Environment = var.environment
      Project     = var.project_name
      ManagedBy   = "terraform"
      Owner       = "platform-team"
    }
  }
}

# Specify required Terraform and provider versions
terraform {
  required_version = ">= 1.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}
```

### Multiple Provider Configurations

For resources that need to be created in different regions:

```hcl
# Default provider configuration
provider "aws" {
  region = "us-east-1"
  alias  = "primary"
  # ... other configuration ...
}

# Secondary region provider
provider "aws" {
  region = "us-west-2"
  alias  = "secondary"
  # ... other configuration ...
}

# Use the specific provider for resources
resource "aws_s3_bucket" "primary_bucket" {
  provider = aws.primary
  bucket   = "my-primary-bucket"
  # ... other configuration ...
}

resource "aws_s3_bucket" "dr_bucket" {
  provider = aws.secondary
  bucket   = "my-dr-bucket"
  # ... other configuration ...
}
```

## Authentication Methods

### Environment Variables (Recommended for CI/CD)

Set these environment variables before running Terraform:

```bash
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_SESSION_TOKEN="your-session-token"  # If using temporary credentials
```

### Shared Credentials File

```hcl
provider "aws" {
  region                   = var.aws_region
  shared_credentials_files = ["~/.aws/credentials"]
  profile                  = "my-profile"
}
```

### Assume Role (Recommended for Cross-Account Access)

```hcl
provider "aws" {
  region = var.aws_region

  assume_role {
    role_arn     = "arn:aws:iam::123456789012:role/TerraformExecutionRole"
    session_name = "terraform-session"
    external_id  = var.external_id  # Optional but recommended for third-party access
  }
}
```

### Web Identity Federation

```hcl
provider "aws" {
  region = var.aws_region

  assume_role_with_web_identity {
    role_arn                = "arn:aws:iam::123456789012:role/TerraformWebIdentityRole"
    session_name            = "terraform-session"
    web_identity_token_file = "/path/to/token/file"
  }
}
```

## State Management

### Remote State Configuration

```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state-bucket"
    key            = "path/to/my/project.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

### DynamoDB State Locking

```hcl
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name = "Terraform State Lock Table"
  }
}
```

### State File Access Control

```hcl
resource "aws_s3_bucket" "terraform_state" {
  bucket = "terraform-state-bucket"

  # Enable versioning to keep history of state files
  versioning {
    enabled = true
  }

  # Enable server-side encryption
  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
}

# Block public access
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Bucket policy to enforce encryption and secure transport
resource "aws_s3_bucket_policy" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Deny"
        Principal = "*"
        Action = "s3:*"
        Resource = [
          "${aws_s3_bucket.terraform_state.arn}",
          "${aws_s3_bucket.terraform_state.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport": "false"
          }
        }
      }
    ]
  })
}
```

## Provider Versioning

### Version Constraints

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.16"  # Allows 4.16.x but not 5.x
    }
  }
}
```

### Version Constraint Types

- `=` - Exact version match
- `!=` - Version not equal to
- `>`, `>=`, `<`, `<=` - Version comparison
- `~>` - Pessimistic constraint (allows only the rightmost version component to increment)

### Version Upgrade Strategy

1. **Pin exact versions in production**: Use `= 4.16.0` for production environments
2. **Use pessimistic constraints in development**: Use `~> 4.16` for development
3. **Test provider upgrades in isolation**: Create a separate branch for provider upgrades
4. **Review changelog**: Always review the provider changelog before upgrading

## Multi-Region Deployments

### Module-based Approach

```hcl
module "us_east_1_deployment" {
  source = "./modules/api_gateway_lambda"

  providers = {
    aws = aws.us_east_1
  }

  environment = "prod"
  region_name = "us-east-1"
  # ... other variables ...
}

module "us_west_2_deployment" {
  source = "./modules/api_gateway_lambda"

  providers = {
    aws = aws.us_west_2
  }

  environment = "prod"
  region_name = "us-west-2"
  # ... other variables ...
}
```

### Regional Resource Naming

```hcl
locals {
  region_code = {
    "us-east-1" = "use1"
    "us-west-2" = "usw2"
    # ... other regions ...
  }

  resource_prefix = "${var.project_name}-${var.environment}-${lookup(local.region_code, var.aws_region)}"
}

resource "aws_lambda_function" "example" {
  function_name = "${local.resource_prefix}-function"
  # ... other configuration ...
}
```

## Multi-Account Deployments

### Provider Configuration for Multiple Accounts

```hcl
provider "aws" {
  alias  = "dev"
  region = var.aws_region

  assume_role {
    role_arn     = "arn:aws:iam::${var.dev_account_id}:role/TerraformExecutionRole"
    session_name = "terraform-dev-session"
  }
}

provider "aws" {
  alias  = "prod"
  region = var.aws_region

  assume_role {
    role_arn     = "arn:aws:iam::${var.prod_account_id}:role/TerraformExecutionRole"
    session_name = "terraform-prod-session"
  }
}
```

### Cross-Account Resource Access

```hcl
# Create a role in the production account that can be assumed by the Lambda in the dev account
resource "aws_iam_role" "cross_account_role" {
  provider = aws.prod
  name     = "CrossAccountAccessRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        AWS = "arn:aws:iam::${var.dev_account_id}:role/LambdaExecutionRole"
      }
    }]
  })
}

# Grant permissions to the role
resource "aws_iam_role_policy" "cross_account_policy" {
  provider = aws.prod
  name     = "CrossAccountAccessPolicy"
  role     = aws_iam_role.cross_account_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action   = ["s3:GetObject", "s3:ListBucket"]
      Effect   = "Allow"
      Resource = [
        "arn:aws:s3:::${aws_s3_bucket.prod_bucket.bucket}",
        "arn:aws:s3:::${aws_s3_bucket.prod_bucket.bucket}/*"
      ]
    }]
  })
}
```

## Workspace Management

### Workspace Configuration

```hcl
# Define workspace-specific variables
locals {
  workspace_config = {
    default = {
      aws_region  = "us-east-1"
      environment = "dev"
      instance_type = "t3.micro"
    }
    staging = {
      aws_region  = "us-east-1"
      environment = "staging"
      instance_type = "t3.small"
    }
    production = {
      aws_region  = "us-east-1"
      environment = "prod"
      instance_type = "t3.medium"
    }
  }

  # Use workspace-specific configuration or default to "default" workspace
  config = lookup(local.workspace_config, terraform.workspace, local.workspace_config["default"])
}

provider "aws" {
  region = local.config.aws_region
}

# Use the workspace-specific configuration
resource "aws_instance" "example" {
  instance_type = local.config.instance_type

  tags = {
    Environment = local.config.environment
  }
}
```

### Workspace-specific Backend Configuration

```bash
# dev.backend.hcl
bucket         = "terraform-state-bucket"
key            = "dev/terraform.tfstate"
region         = "us-east-1"
dynamodb_table = "terraform-locks"
encrypt        = true

# prod.backend.hcl
bucket         = "terraform-state-bucket"
key            = "prod/terraform.tfstate"
region         = "us-east-1"
dynamodb_table = "terraform-locks"
encrypt        = true
```

Initialize with specific backend configuration:

```bash
terraform init -backend-config=dev.backend.hcl
```

## Security Best Practices

### Least Privilege IAM Policies

Create a dedicated IAM role for Terraform with only the permissions needed:

```hcl
resource "aws_iam_policy" "terraform_policy" {
  name        = "TerraformExecutionPolicy"
  description = "Policy for Terraform execution"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = [
          "lambda:*",
          "apigateway:*",
          "iam:GetRole",
          "iam:PassRole",
          "iam:CreateRole",
          "iam:DeleteRole",
          "iam:PutRolePolicy",
          "iam:DeleteRolePolicy",
          "logs:*",
          "cloudwatch:*",
          "s3:*"
        ]
        Resource = "*"
      }
    ]
  })
}

resource "aws_iam_role" "terraform_role" {
  name = "TerraformExecutionRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        AWS = "arn:aws:iam::${var.admin_account_id}:root"
      }
      Condition = {
        StringEquals = {
          "sts:ExternalId": var.external_id
        }
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "terraform_policy_attachment" {
  role       = aws_iam_role.terraform_role.name
  policy_arn = aws_iam_policy.terraform_policy.arn
}
```

### Sensitive Data Handling

1. **Use environment variables for credentials**:
   ```bash
   export TF_VAR_database_password="supersecretpassword"
   ```

2. **Mark variables as sensitive**:
   ```hcl
   variable "database_password" {
     description = "Password for the database"
     type        = string
     sensitive   = true
   }
   ```

3. **Use AWS Secrets Manager or Parameter Store**:
   ```hcl
   data "aws_secretsmanager_secret" "db_password" {
     name = "db/password"
   }

   data "aws_secretsmanager_secret_version" "db_password" {
     secret_id = data.aws_secretsmanager_secret.db_password.id
   }

   locals {
     db_password = jsondecode(data.aws_secretsmanager_secret_version.db_password.secret_string)["password"]
   }
   ```

### Provider Configuration Security

1. **Never hardcode credentials in provider blocks**
2. **Use assume_role for cross-account access**
3. **Implement MFA when possible**:
   ```hcl
   provider "aws" {
     region = var.aws_region

     assume_role {
       role_arn    = var.role_arn
       session_name = "terraform-session"
       external_id  = var.external_id

       # MFA configuration
       serial_number = var.mfa_serial
       token_code    = var.mfa_token
     }
   }
   ```

---

For more information on integrating with other AWS services, refer to the other documents in this repository:
- [API Gateway Guidelines](api_gateway.md)
- [Lambda Guidelines](lambda.md)
- [IAM Best Practices](iam.md)
