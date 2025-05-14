# CI/CD Integration Best Practices for Terraform AWS API Gateway and Lambda

This document outlines the best practices for implementing CI/CD (Continuous Integration/Continuous Deployment) pipelines for AWS API Gateway and Lambda projects using Terraform. Following these guidelines will help ensure your infrastructure deployments are automated, consistent, and reliable.

## Table of Contents

- [CI/CD Pipeline Overview](#cicd-pipeline-overview)
- [Source Control Best Practices](#source-control-best-practices)
- [Terraform State Management](#terraform-state-management)
- [Environment Management](#environment-management)
- [Security Considerations](#security-considerations)
- [Testing Strategies](#testing-strategies)
- [Deployment Strategies](#deployment-strategies)
- [Pipeline Implementation Examples](#pipeline-implementation-examples)
- [Monitoring and Rollbacks](#monitoring-and-rollbacks)

## CI/CD Pipeline Overview

A well-designed CI/CD pipeline for Terraform-managed AWS API Gateway and Lambda projects should include the following stages:

1. **Source**: Code is pushed to a repository
2. **Validate**: Terraform code is validated and linted
3. **Plan**: Terraform plan is generated and reviewed
4. **Test**: Infrastructure and application tests are run
5. **Apply**: Terraform changes are applied to the target environment
6. **Verify**: Deployment is verified through post-deployment tests
7. **Monitor**: Deployment is monitored for issues

### Pipeline Architecture Diagram

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│         │    │         │    │         │    │         │    │         │    │         │    │         │
│ Source  │───►│Validate │───►│  Plan   │───►│  Test   │───►│  Apply  │───►│ Verify  │───►│ Monitor │
│         │    │         │    │         │    │         │    │         │    │         │    │         │
└─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘
```

## Source Control Best Practices

### Repository Structure

Organize your repository to separate application code from infrastructure code:

```
project/
├── README.md
├── .gitignore
├── terraform/
│   ├── environments/
│   │   ├── dev/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── terraform.tfvars
│   │   ├── staging/
│   │   └── prod/
│   ├── modules/
│   │   ├── api_gateway/
│   │   ├── lambda/
│   │   └── monitoring/
│   └── shared/
│       ├── provider.tf
│       └── backend.tf
├── src/
│   ├── functions/
│   │   ├── function1/
│   │   │   ├── index.js
│   │   │   ├── package.json
│   │   │   └── tests/
│   │   └── function2/
│   └── layers/
│       └── common/
├── tests/
│   ├── integration/
│   └── e2e/
└── ci/
    ├── Dockerfile
    ├── scripts/
    └── pipelines/
        ├── azure-pipelines.yml
        ├── github-actions.yml
        └── buildspec.yml
```

### Branching Strategy

Implement a branching strategy that aligns with your deployment environments:

#### Git Flow

```
master (production)
  │
  ├── release branches (staging)
  │     │
  │     └── feature branches (development)
  │
  └── hotfix branches (urgent fixes)
```

#### Trunk-Based Development

```
main (trunk)
  │
  ├── short-lived feature branches
  │
  └── environment branches (if needed)
```

### Commit Message Conventions

Use conventional commits to make your history more readable and enable automated versioning:

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

Types:
- `feat`: A new feature
- `fix`: A bug fix
- `chore`: Routine tasks, maintenance
- `docs`: Documentation changes
- `style`: Code style changes (formatting, etc.)
- `refactor`: Code changes that neither fix bugs nor add features
- `test`: Adding or correcting tests
- `infra`: Infrastructure changes

Example:
```
feat(lambda): add new payment processing function

- Implements Stripe integration
- Adds error handling for declined payments

Closes #123
```

## Terraform State Management

### Remote State Configuration

Store Terraform state in a remote backend with proper locking:

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "terraform-state-${var.project_name}"
    key            = "${var.environment}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

### State Isolation

Isolate state by environment to prevent cross-environment impacts:

```bash
# CI/CD script example
#!/bin/bash
set -e

ENVIRONMENT=$1

# Initialize with the appropriate backend configuration
terraform init \
  -backend-config="key=${ENVIRONMENT}/terraform.tfstate" \
  -backend-config="bucket=terraform-state-${PROJECT_NAME}" \
  -backend-config="dynamodb_table=terraform-locks"

# Plan and apply
terraform plan -var-file="environments/${ENVIRONMENT}/terraform.tfvars" -out=tfplan
terraform apply tfplan
```

### State Access Control

Implement proper access controls for your Terraform state:

```hcl
# IAM policy for CI/CD service role
resource "aws_iam_policy" "terraform_state_access" {
  name        = "terraform-state-access"
  description = "Policy for accessing Terraform state"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:ListBucket"
        ]
        Resource = [
          "arn:aws:s3:::terraform-state-${var.project_name}",
          "arn:aws:s3:::terraform-state-${var.project_name}/*"
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:DeleteItem"
        ]
        Resource = "arn:aws:dynamodb:*:*:table/terraform-locks"
      }
    ]
  })
}
```

## Environment Management

### Environment Configuration

Define environment-specific variables in separate files:

```hcl
# environments/dev/terraform.tfvars
environment         = "dev"
aws_region          = "us-east-1"
lambda_memory_size  = 128
log_retention_days  = 7
enable_xray_tracing = true
```

```hcl
# environments/prod/terraform.tfvars
environment         = "prod"
aws_region          = "us-east-1"
lambda_memory_size  = 512
log_retention_days  = 90
enable_xray_tracing = true
```

### Environment Promotion

Implement a promotion strategy for changes across environments:

```bash
# CI/CD script for environment promotion
#!/bin/bash
set -e

SOURCE_ENV=$1
TARGET_ENV=$2

# Export the state from source environment
terraform init -backend-config="key=${SOURCE_ENV}/terraform.tfstate"
terraform state pull > ${SOURCE_ENV}.tfstate

# Import the state to target environment with target-specific variables
terraform init -backend-config="key=${TARGET_ENV}/terraform.tfstate"
terraform plan -var-file="environments/${TARGET_ENV}/terraform.tfvars" -out=tfplan
terraform apply tfplan
```

### Environment-Specific CI/CD Workflows

Configure different CI/CD workflows for different environments:

```yaml
# GitHub Actions example
name: Terraform CI/CD

on:
  push:
    branches:
      - main
      - 'release/**'
  pull_request:
    branches:
      - main

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v1
      - name: Terraform Format
        run: terraform fmt -check -recursive
      - name: Terraform Validate
        run: |
          cd terraform
          terraform init -backend=false
          terraform validate

  plan-dev:
    if: github.event_name == 'pull_request'
    needs: validate
    runs-on: ubuntu-latest
    steps:
      # ... steps to plan changes for dev environment

  apply-dev:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    needs: validate
    runs-on: ubuntu-latest
    steps:
      # ... steps to apply changes to dev environment

  apply-staging:
    if: startsWith(github.ref, 'refs/heads/release/')
    needs: validate
    runs-on: ubuntu-latest
    steps:
      # ... steps to apply changes to staging environment

  apply-prod:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    needs: [apply-dev, manual-approval]
    runs-on: ubuntu-latest
    steps:
      # ... steps to apply changes to production environment
```

## Security Considerations

### Secrets Management

Never store sensitive information in your Terraform code or repository. Use a secrets manager:

```hcl
# Retrieve secrets from AWS Secrets Manager
data "aws_secretsmanager_secret" "database_credentials" {
  name = "${var.project_name}/${var.environment}/db-credentials"
}

data "aws_secretsmanager_secret_version" "database_credentials" {
  secret_id = data.aws_secretsmanager_secret.database_credentials.id
}

locals {
  db_credentials = jsondecode(data.aws_secretsmanager_secret_version.database_credentials.secret_string)
}

# Use in Lambda environment variables
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

### CI/CD Service Account Permissions

Use the principle of least privilege for your CI/CD service accounts:

```hcl
# IAM policy for CI/CD service role
resource "aws_iam_policy" "cicd_service_policy" {
  name        = "${var.project_name}-cicd-service-policy"
  description = "Policy for CI/CD service to deploy infrastructure"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      # Lambda permissions
      {
        Effect = "Allow"
        Action = [
          "lambda:GetFunction",
          "lambda:CreateFunction",
          "lambda:DeleteFunction",
          "lambda:UpdateFunctionCode",
          "lambda:UpdateFunctionConfiguration",
          "lambda:ListVersionsByFunction",
          "lambda:PublishVersion",
          "lambda:CreateAlias",
          "lambda:UpdateAlias",
          "lambda:DeleteAlias"
        ]
        Resource = "arn:aws:lambda:${var.aws_region}:${data.aws_caller_identity.current.account_id}:function:${var.project_name}-*"
      },
      # API Gateway permissions
      {
        Effect = "Allow"
        Action = [
          "apigateway:GET",
          "apigateway:POST",
          "apigateway:PUT",
          "apigateway:DELETE",
          "apigateway:PATCH"
        ]
        Resource = [
          "arn:aws:apigateway:${var.aws_region}::/restapis",
          "arn:aws:apigateway:${var.aws_region}::/restapis/*"
        ]
      },
      # IAM permissions (restricted to project-specific roles)
      {
        Effect = "Allow"
        Action = [
          "iam:GetRole",
          "iam:CreateRole",
          "iam:DeleteRole",
          "iam:PutRolePolicy",
          "iam:DeleteRolePolicy",
          "iam:PassRole"
        ]
        Resource = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:role/${var.project_name}-*"
      }
      # ... other necessary permissions
    ]
  })
}
```

### Security Scanning

Integrate security scanning into your CI/CD pipeline:

```yaml
# GitHub Actions example with security scanning
jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Terraform security scan
        uses: triat/terraform-security-scan@v3
        with:
          tfsec_args: --exclude-downloaded-modules

      - name: Run tfsec with sarif output
        uses: aquasecurity/tfsec-sarif-action@v0.1.0
        with:
          sarif_file: tfsec.sarif

      - name: Upload SARIF file
        uses: github/codeql-action/upload-sarif@v1
        with:
          sarif_file: tfsec.sarif
```

## Testing Strategies

### Unit Testing

Test individual Terraform modules:

```go
// Example using Terratest
package test

import (
  "testing"
  "github.com/gruntwork-io/terratest/modules/terraform"
  "github.com/stretchr/testify/assert"
)

func TestLambdaModule(t *testing.T) {
  terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
    TerraformDir: "../modules/lambda",
    Vars: map[string]interface{}{
      "function_name": "test-function",
      "runtime": "nodejs14.x",
      "handler": "index.handler",
      "memory_size": 128,
    },
  })

  defer terraform.Destroy(t, terraformOptions)
  terraform.InitAndApply(t, terraformOptions)

  functionName := terraform.Output(t, terraformOptions, "function_name")
  assert.Equal(t, "test-function", functionName)
}
```

### Integration Testing

Test the interaction between different infrastructure components:

```bash
#!/bin/bash
# Integration test script

# Deploy test infrastructure
cd terraform/environments/test
terraform init
terraform apply -auto-approve

# Get API endpoint from Terraform output
API_ENDPOINT=$(terraform output -raw api_endpoint)

# Test API endpoints
echo "Testing GET /items endpoint..."
RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" $API_ENDPOINT/items)
if [ "$RESPONSE" -ne 200 ]; then
  echo "Test failed: Expected 200, got $RESPONSE"
  exit 1
fi

echo "Testing POST /items endpoint..."
RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" -X POST -H "Content-Type: application/json" -d '{"name":"test"}' $API_ENDPOINT/items)
if [ "$RESPONSE" -ne 201 ]; then
  echo "Test failed: Expected 201, got $RESPONSE"
  exit 1
fi

# Clean up test infrastructure
terraform destroy -auto-approve
```

### End-to-End Testing

Test the entire system from a user's perspective:

```javascript
// Example using Jest and Supertest
const request = require('supertest');
const apiEndpoint = process.env.API_ENDPOINT;

describe('API Gateway and Lambda Integration', () => {
  test('GET /items returns 200 and items array', async () => {
    const response = await request(apiEndpoint)
      .get('/items')
      .set('Accept', 'application/json');

    expect(response.status).toBe(200);
    expect(Array.isArray(response.body)).toBe(true);
  });

  test('POST /items creates a new item', async () => {
    const newItem = { name: 'Test Item', description: 'Created during E2E test' };

    const response = await request(apiEndpoint)
      .post('/items')
      .send(newItem)
      .set('Accept', 'application/json');

    expect(response.status).toBe(201);
    expect(response.body).toHaveProperty('id');
    expect(response.body.name).toBe(newItem.name);
  });
});
```

## Deployment Strategies

### Blue-Green Deployments

Implement blue-green deployments for API Gateway and Lambda:

```hcl
# Lambda version and alias for blue-green deployment
resource "aws_lambda_function_version" "example_version" {
  function_name = aws_lambda_function.example.function_name
  description   = "Version ${var.app_version}"
}

resource "aws_lambda_alias" "live" {
  name             = "live"
  function_name    = aws_lambda_function.example.function_name
  function_version = aws_lambda_function_version.example_version.version
}

# API Gateway stage variables to point to the current live alias
resource "aws_api_gateway_stage" "example" {
  # ... other configuration ...

  variables = {
    lambdaAlias = "live"
  }
}

# Lambda permission for API Gateway to invoke the aliased function
resource "aws_lambda_permission" "api_gateway" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.example.function_name
  qualifier     = aws_lambda_alias.live.name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_api_gateway_rest_api.example.execution_arn}/*/*"
}
```

### Canary Deployments

Implement canary deployments for Lambda functions:

```hcl
# Lambda alias with traffic shifting for canary deployment
resource "aws_lambda_alias" "live" {
  name             = "live"
  function_name    = aws_lambda_function.example.function_name
  function_version = aws_lambda_function_version.current.version

  # Shift 10% of traffic to the new version
  routing_config {
    additional_version_weights = {
      "${aws_lambda_function_version.new.version}" = 0.1
    }
  }
}

# CloudWatch alarm to monitor the new version
resource "aws_cloudwatch_metric_alarm" "canary_errors" {
  alarm_name          = "${aws_lambda_function.example.function_name}-canary-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  period              = 60
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "This metric monitors errors in the canary deployment"

  dimensions = {
    FunctionName = aws_lambda_function.example.function_name
    Resource     = "${aws_lambda_function.example.function_name}:${aws_lambda_function_version.new.version}"
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

### Rollback Strategy

Implement automated rollbacks in case of deployment failures:

```bash
#!/bin/bash
# Rollback script for failed deployments

# Get the current and previous version
CURRENT_VERSION=$(aws lambda get-alias --function-name $FUNCTION_NAME --name live --query 'FunctionVersion' --output text)
PREVIOUS_VERSION=$(aws lambda list-versions-by-function --function-name $FUNCTION_NAME --query "Versions[?Version!='$CURRENT_VERSION']|[?Version!='$LATEST'].Version" --output text | sort -rn | head -1)

# Update the alias to point to the previous version
aws lambda update-alias --function-name $FUNCTION_NAME --name live --function-version $PREVIOUS_VERSION

echo "Rolled back $FUNCTION_NAME from version $CURRENT_VERSION to $PREVIOUS_VERSION"
```

## Pipeline Implementation Examples

### GitHub Actions

```yaml
# .github/workflows/terraform.yml
name: 'Terraform CI/CD'

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  terraform:
    name: 'Terraform'
    runs-on: ubuntu-latest

    # Use the Bash shell regardless whether the GitHub Actions runner is ubuntu-latest, macos-latest, or windows-latest
    defaults:
      run:
        shell: bash
        working-directory: ./terraform

    steps:
    # Checkout the repository to the GitHub Actions runner
    - name: Checkout
      uses: actions/checkout@v2

    # Install the latest version of Terraform CLI
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v1

    # Initialize a new or existing Terraform working directory by creating initial files, loading any remote state, downloading modules, etc.
    - name: Terraform Init
      run: terraform init
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

    # Checks that all Terraform configuration files adhere to a canonical format
    - name: Terraform Format
      run: terraform fmt -check

    # Validates the configuration files in a directory, referring only to the configuration and not accessing any remote services
    - name: Terraform Validate
      run: terraform validate

    # Generate an execution plan for Terraform
    - name: Terraform Plan
      run: terraform plan -var-file="environments/dev/terraform.tfvars"
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

    # Apply the Terraform configuration (only on push to main)
    - name: Terraform Apply
      if: github.ref == 'refs/heads/main' && github.event_name == 'push'
      run: terraform apply -auto-approve -var-file="environments/dev/terraform.tfvars"
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### AWS CodePipeline

```hcl
# AWS CodePipeline for Terraform deployment
resource "aws_codepipeline" "terraform_pipeline" {
  name     = "${var.project_name}-terraform-pipeline"
  role_arn = aws_iam_role.codepipeline_role.arn

  artifact_store {
    location = aws_s3_bucket.artifact_bucket.bucket
    type     = "S3"
  }

  stage {
    name = "Source"

    action {
      name             = "Source"
      category         = "Source"
      owner            = "AWS"
      provider         = "CodeStarSourceConnection"
      version          = "1"
      output_artifacts = ["source_output"]

      configuration = {
        ConnectionArn    = aws_codestarconnections_connection.github.arn
        FullRepositoryId = "${var.github_owner}/${var.github_repo}"
        BranchName       = var.github_branch
      }
    }
  }

  stage {
    name = "Validate"

    action {
      name             = "Validate"
      category         = "Build"
      owner            = "AWS"
      provider         = "CodeBuild"
      input_artifacts  = ["source_output"]
      output_artifacts = ["validate_output"]
      version          = "1"

      configuration = {
        ProjectName = aws_codebuild_project.terraform_validate.name
      }
    }
  }

  stage {
    name = "Plan"

    action {
      name             = "Plan"
      category         = "Build"
      owner            = "AWS"
      provider         = "CodeBuild"
      input_artifacts  = ["validate_output"]
      output_artifacts = ["plan_output"]
      version          = "1"

      configuration = {
        ProjectName = aws_codebuild_project.terraform_plan.name
      }
    }
  }

  stage {
    name = "Approve"

    action {
      name     = "Approval"
      category = "Approval"
      owner    = "AWS"
      provider = "Manual"
      version  = "1"
    }
  }

  stage {
    name = "Apply"

    action {
      name            = "Apply"
      category        = "Build"
      owner           = "AWS"
      provider        = "CodeBuild"
      input_artifacts = ["plan_output"]
      version         = "1"

      configuration = {
        ProjectName = aws_codebuild_project.terraform_apply.name
      }
    }
  }
}
```

### Azure DevOps

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
    - main
    - 'release/*'

pool:
  vmImage: 'ubuntu-latest'

variables:
  - group: terraform-secrets
  - name: TF_WORKING_DIR
    value: 'terraform'

stages:
- stage: Validate
  jobs:
  - job: ValidateTerraform
    steps:
    - task: TerraformInstaller@0
      inputs:
        terraformVersion: '1.0.0'

    - task: TerraformTaskV2@2
      displayName: 'Terraform Init'
      inputs:
        provider: 'aws'
        command: 'init'
        workingDirectory: '$(TF_WORKING_DIR)'
        backendServiceAWS: 'AWS-Service-Connection'
        backendAWSBucketName: 'terraform-state-bucket'
        backendAWSKey: 'dev/terraform.tfstate'

    - task: TerraformTaskV2@2
      displayName: 'Terraform Validate'
      inputs:
        provider: 'aws'
        command: 'validate'
        workingDirectory: '$(TF_WORKING_DIR)'

    - task: TerraformTaskV2@2
      displayName: 'Terraform Plan'
      inputs:
        provider: 'aws'
        command: 'plan'
        workingDirectory: '$(TF_WORKING_DIR)'
        environmentServiceNameAWS: 'AWS-Service-Connection'
        commandOptions: '-var-file="environments/dev/terraform.tfvars" -out=tfplan'

- stage: Deploy
  dependsOn: Validate
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: DeployTerraform
    environment: 'dev'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: TerraformInstaller@0
            inputs:
              terraformVersion: '1.0.0'

          - task: TerraformTaskV2@2
            displayName: 'Terraform Init'
            inputs:
              provider: 'aws'
              command: 'init'
              workingDirectory: '$(TF_WORKING_DIR)'
              backendServiceAWS: 'AWS-Service-Connection'
              backendAWSBucketName: 'terraform-state-bucket'
              backendAWSKey: 'dev/terraform.tfstate'

          - task: TerraformTaskV2@2
            displayName: 'Terraform Apply'
            inputs:
              provider: 'aws'
              command: 'apply'
              workingDirectory: '$(TF_WORKING_DIR)'
              environmentServiceNameAWS: 'AWS-Service-Connection'
              commandOptions: '-var-file="environments/dev/terraform.tfvars" -auto-approve'
```

## Monitoring and Rollbacks

### Post-Deployment Verification

Implement post-deployment verification to ensure the deployment was successful:

```bash
#!/bin/bash
# Post-deployment verification script

# Get API endpoint from Terraform output
API_ENDPOINT=$(terraform output -raw api_endpoint)

# Test critical endpoints
echo "Verifying API endpoints..."
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" $API_ENDPOINT/health)

if [ $HTTP_STATUS -ne 200 ]; then
  echo "Health check failed with status $HTTP_STATUS"
  exit 1
fi

# Check Lambda function health
FUNCTION_NAME=$(terraform output -raw lambda_function_name)
INVOCATION_RESULT=$(aws lambda invoke --function-name $FUNCTION_NAME --payload '{"action": "health"}' /tmp/lambda-result.json)
ERROR_STATUS=$(echo $INVOCATION_RESULT | jq -r .FunctionError)

if [ "$ERROR_STATUS" != "null" ]; then
  echo "Lambda health check failed: $ERROR_STATUS"
  cat /tmp/lambda-result.json
  exit 1
fi

echo "Post-deployment verification successful"
```

### Automated Rollbacks

Set up automated rollbacks in case of deployment failures:

```yaml
# GitHub Actions example with rollback
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v1

      - name: Terraform Init
