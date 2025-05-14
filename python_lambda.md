---
title: "Python Lambda Guidelines for AWS"
category: "Lambda"
tags: ["python", "aws", "terraform", "serverless"]
last_updated: "2025-05-14"
---

# Python Lambda Guidelines for AWS

This document outlines the best practices for developing and deploying AWS Lambda functions using Python and Terraform. These guidelines are specifically tailored for EpsiHack's Python-based serverless applications.

## Table of Contents

- [Development Environment Setup](#development-environment-setup)
- [Project Structure](#project-structure)
- [Handler Implementation](#handler-implementation)
- [Dependency Management](#dependency-management)
- [Building and Packaging](#building-and-packaging)
- [Testing](#testing)
- [Deployment with Terraform](#deployment-with-terraform)
- [Performance Optimization](#performance-optimization)
- [Logging and Monitoring](#logging-and-monitoring)
- [Common Patterns and Examples](#common-patterns-and-examples)

## Development Environment Setup

### Required Tools

- Python 3.9+ (recommended)
- pip and virtualenv/venv
- AWS CLI
- AWS SAM CLI (for local testing)
- IDE with Python support (PyCharm, VS Code, etc.)

### Virtual Environment Setup

```bash
# Create a virtual environment
python -m venv venv

# Activate the virtual environment
# On Windows
venv\Scripts\activate
# On macOS/Linux
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt
```

### AWS SDK Installation

```bash
# Install AWS SDK for Python (Boto3)
pip install boto3

# Install AWS Lambda Powertools (recommended)
pip install aws-lambda-powertools
```

## Project Structure

### Recommended Structure

```
project/
├── src/
│   ├── handlers/
│   │   ├── __init__.py
│   │   ├── api_handler.py
│   │   └── event_handler.py
│   ├── models/
│   │   ├── __init__.py
│   │   └── user.py
│   ├── services/
│   │   ├── __init__.py
│   │   └── database_service.py
│   └── utils/
│       ├── __init__.py
│       └── helpers.py
├── tests/
│   ├── __init__.py
│   ├── unit/
│   │   ├── __init__.py
│   │   └── test_api_handler.py
│   └── integration/
│       ├── __init__.py
│       └── test_database_service.py
├── requirements.txt
├── requirements-dev.txt
├── setup.py
├── README.md
└── terraform/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

### Python Package Structure

```python
# setup.py
from setuptools import setup, find_packages

setup(
    name="epsihack-lambda",
    version="0.1.0",
    packages=find_packages(where="src"),
    package_dir={"": "src"},
    install_requires=[
        "boto3>=1.26.0",
        "aws-lambda-powertools>=2.13.0",
    ],
)
```

## Handler Implementation

### Basic Lambda Handler

```python
# src/handlers/api_handler.py
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """
    Basic Lambda handler function

    Parameters:
    - event: The event data
    - context: The Lambda context object

    Returns:
    - API Gateway response object
    """
    logger.info(f"Received event: {json.dumps(event)}")

    try:
        # Process the event
        result = process_event(event)

        # Return successful response
        return {
            'statusCode': 200,
            'headers': {
                'Content-Type': 'application/json'
            },
            'body': json.dumps({
                'message': 'Success',
                'data': result
            })
        }
    except Exception as e:
        logger.error(f"Error processing event: {str(e)}")

        # Return error response
        return {
            'statusCode': 500,
            'headers': {
                'Content-Type': 'application/json'
            },
            'body': json.dumps({
                'message': 'Error processing request',
                'error': str(e)
            })
        }

def process_event(event):
    # Implement your business logic here
    return {"processed": True}
```

### Using AWS Lambda Powertools

```python
# src/handlers/powertools_handler.py
import json
from aws_lambda_powertools import Logger, Metrics, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.utilities.typing import LambdaContext
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.metrics import MetricUnit

# Initialize Powertools
logger = Logger()
tracer = Tracer()
metrics = Metrics()
app = APIGatewayRestResolver()

@app.get("/users")
def get_users():
    # Log with structured keys
    logger.info("Fetching all users")

    # Add custom metrics
    metrics.add_metric(name="UsersRetrieved", unit=MetricUnit.Count, value=1)

    # Return response
    return {"users": ["user1", "user2"]}

@app.get("/users/<user_id>")
def get_user(user_id):
    # Log with structured keys
    logger.info(f"Fetching user", extra={"user_id": user_id})

    # Add custom metrics
    metrics.add_metric(name="UserRetrieved", unit=MetricUnit.Count, value=1)

    # Return response
    return {"user_id": user_id, "name": "Example User"}

@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
@metrics.log_metrics(capture_cold_start_metric=True)
def lambda_handler(event, context: LambdaContext):
    try:
        return app.resolve(event, context)
    except Exception as e:
        logger.exception("Error handling request")
        return {
            "statusCode": 500,
            "body": json.dumps({"error": "Internal Server Error"})
        }
```

## Dependency Management

### Requirements File

```
# requirements.txt
boto3==1.26.135
aws-lambda-powertools==2.13.0
requests==2.28.2
pydantic==1.10.7
```

### Development Requirements

```
# requirements-dev.txt
-r requirements.txt
pytest==7.3.1
pytest-mock==3.10.0
moto==4.1.6
black==23.3.0
isort==5.12.0
flake8==6.0.0
mypy==1.2.0
```

### Layer Management

Consider using Lambda Layers for common dependencies:

```python
# src/handlers/layer_handler.py
# These dependencies would be in a layer
import boto3
import requests
from aws_lambda_powertools import Logger

logger = Logger()

def lambda_handler(event, context):
    # Your code using the dependencies from the layer
    response = requests.get("https://api.example.com/data")
    return {"statusCode": 200, "body": response.text}
```

## Building and Packaging

### Creating a Deployment Package

```bash
# Create a deployment package
pip install --target ./package -r requirements.txt
cd package
zip -r ../lambda_function.zip .
cd ..
zip -g lambda_function.zip -r src
```

### Using Docker for Packaging

```dockerfile
# Dockerfile
FROM public.ecr.aws/lambda/python:3.9

# Copy requirements file
COPY requirements.txt ${LAMBDA_TASK_ROOT}

# Install dependencies
RUN pip install -r requirements.txt

# Copy function code
COPY src/ ${LAMBDA_TASK_ROOT}/

# Set the handler
CMD ["handlers.api_handler.lambda_handler"]
```

```bash
# Build and push the Docker image
docker build -t my-lambda-image .
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
docker tag my-lambda-image:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-lambda-image:latest
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-lambda-image:latest
```

## Testing

### Unit Testing

```python
# tests/unit/test_api_handler.py
import json
import pytest
from src.handlers.api_handler import lambda_handler, process_event

def test_process_event():
    # Test the business logic function
    result = process_event({"test": "data"})
    assert result == {"processed": True}

def test_lambda_handler_success():
    # Mock event and context
    event = {"test": "data"}
    context = {}

    # Call the handler
    response = lambda_handler(event, context)

    # Verify the response
    assert response["statusCode"] == 200
    body = json.loads(response["body"])
    assert body["message"] == "Success"
    assert body["data"] == {"processed": True}

def test_lambda_handler_error():
    # Mock event and context
    event = {"test": "data"}
    context = {}

    # Mock process_event to raise an exception
    def mock_process_event(event):
        raise Exception("Test error")

    # Patch the process_event function
    import src.handlers.api_handler
    original_process_event = src.handlers.api_handler.process_event
    src.handlers.api_handler.process_event = mock_process_event

    try:
        # Call the handler
        response = lambda_handler(event, context)

        # Verify the response
        assert response["statusCode"] == 500
        body = json.loads(response["body"])
        assert body["message"] == "Error processing request"
        assert body["error"] == "Test error"
    finally:
        # Restore the original function
        src.handlers.api_handler.process_event = original_process_event
```

### Integration Testing with Moto

```python
# tests/integration/test_dynamodb_integration.py
import boto3
import pytest
from moto import mock_dynamodb

from src.services.database_service import DynamoDBService

@pytest.fixture
def dynamodb_table():
    with mock_dynamodb():
        # Create a mock DynamoDB table
        dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
        table = dynamodb.create_table(
            TableName='users',
            KeySchema=[
                {'AttributeName': 'user_id', 'KeyType': 'HASH'}
            ],
            AttributeDefinitions=[
                {'AttributeName': 'user_id', 'AttributeType': 'S'}
            ],
            ProvisionedThroughput={'ReadCapacityUnits': 5, 'WriteCapacityUnits': 5}
        )
        yield table

def test_save_and_get_item(dynamodb_table):
    # Initialize the service
    service = DynamoDBService(table_name='users')

    # Save an item
    user = {'user_id': '123', 'name': 'Test User', 'email': 'test@example.com'}
    service.save_item(user)

    # Get the item
    retrieved_user = service.get_item('user_id', '123')

    # Verify the item was saved and retrieved correctly
    assert retrieved_user['user_id'] == '123'
    assert retrieved_user['name'] == 'Test User'
    assert retrieved_user['email'] == 'test@example.com'
```

### Local Testing with AWS SAM

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Resources:
  PythonFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./
      Handler: src/handlers/api_handler.lambda_handler
      Runtime: python3.9
      Architectures:
        - x86_64
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /hello
            Method: get
```

```bash
# Build and run locally
sam build
sam local invoke PythonFunction --event events/api-event.json
sam local start-api
```

## Deployment with Terraform

### Lambda Function Resource

```hcl
resource "aws_lambda_function" "python_function" {
  function_name    = "${var.project_name}-${var.environment}-python-function"
  filename         = "../lambda_function.zip"
  source_code_hash = filebase64sha256("../lambda_function.zip")
  handler          = "src.handlers.api_handler.lambda_handler"
  runtime          = "python3.9"
  memory_size      = 256
  timeout          = 30

  role = aws_iam_role.lambda_execution_role.arn

  environment {
    variables = {
      ENVIRONMENT = var.environment
      LOG_LEVEL   = var.environment == "prod" ? "INFO" : "DEBUG"
      POWERTOOLS_SERVICE_NAME = "${var.project_name}-api"
      POWERTOOLS_METRICS_NAMESPACE = "${var.project_name}"
    }
  }

  # Enable X-Ray tracing
  tracing_config {
    mode = "Active"
  }

  # Configure VPC if needed
  vpc_config {
    subnet_ids         = var.subnet_ids
    security_group_ids = [aws_security_group.lambda_sg.id]
  }

  tags = {
    Environment = var.environment
    Project     = var.project_name
    Runtime     = "python3.9"
  }
}
```

### Lambda Layer for Dependencies

```hcl
resource "aws_lambda_layer_version" "python_dependencies" {
  layer_name = "${var.project_name}-${var.environment}-python-dependencies"
  filename   = "../python_dependencies.zip"

  compatible_runtimes = ["python3.9"]

  source_code_hash = filebase64sha256("../python_dependencies.zip")
}

# Reference the layer in your Lambda function
resource "aws_lambda_function" "python_function" {
  # ... other configuration ...
  layers = [aws_lambda_layer_version.python_dependencies.arn]
}
```

### Container Image Deployment

```hcl
resource "aws_lambda_function" "python_container_function" {
  function_name = "${var.project_name}-${var.environment}-python-container"
  package_type  = "Image"
  image_uri     = "${aws_ecr_repository.lambda_repo.repository_url}:latest"

  role = aws_iam_role.lambda_execution_role.arn

  # Configure memory and timeout
  memory_size = 512
  timeout     = 30

  # Enable X-Ray tracing
  tracing_config {
    mode = "Active"
  }

  tags = {
    Environment = var.environment
    Project     = var.project_name
    Runtime     = "python3.9-container"
  }
}
```

## Performance Optimization

### Cold Start Mitigation

1. **Use Provisioned Concurrency**:
   ```hcl
   resource "aws_lambda_provisioned_concurrency_config" "python_function" {
     function_name                     = aws_lambda_function.python_function.function_name
     provisioned_concurrent_executions = 5
     qualifier                         = aws_lambda_alias.python_function_alias.name
   }
   ```

2. **Minimize Package Size**:
   ```bash
   # Use a tool like lambda-packages to reduce size
   pip install lambda-packages

   # Or use a tool like serverless-python-requirements
   npm install --save serverless-python-requirements
   ```

3. **Optimize Imports**:
   ```python
   # Avoid importing at the handler level
   def lambda_handler(event, context):
       # Import only when needed
       import heavy_module
       return heavy_module.process(event)
   ```

### Memory Allocation

```hcl
resource "aws_lambda_function" "python_function" {
  # ... other configuration ...
  memory_size = 1024  # Adjust based on function needs
}
```

### Keep Functions Warm

```python
# src/handlers/warmer.py
def lambda_handler(event, context):
    """
    Handler for warming Lambda functions
    """
    # Check if this is a warming event
    if event.get('warmer', False):
        print("Warming function")
        return {"warmed": True}

    # Regular handler logic
    # ...
```

## Logging and Monitoring

### Structured Logging

```python
# src/handlers/logging_example.py
import json
import logging
import os
import time
import uuid

# Configure logger
logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    # Create a request ID
    request_id = event.get('requestContext', {}).get('requestId', str(uuid.uuid4()))

    # Log with structured data
    logger.info(json.dumps({
        'request_id': request_id,
        'event_type': 'request_received',
        'timestamp': time.time(),
        'environment': os.environ.get('ENVIRONMENT', 'dev'),
        'function_name': context.function_name,
        'path': event.get('path'),
        'method': event.get('httpMethod')
    }))

    # Process the request
    # ...

    # Log completion
    logger.info(json.dumps({
        'request_id': request_id,
        'event_type': 'request_completed',
        'timestamp': time.time(),
        'duration_ms': (time.time() - start_time) * 1000
    }))

    return {
        'statusCode': 200,
        'body': json.dumps({'message': 'Success'})
    }
```

### Using AWS Lambda Powertools for Logging

```python
# src/handlers/powertools_logging.py
from aws_lambda_powertools import Logger
from aws_lambda_powertools.logging import correlation_paths

# Initialize logger
logger = Logger(service="my-service")

@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
def lambda_handler(event, context):
    # Log with structured keys
    logger.info("Processing request", extra={
        "path": event.get("path"),
        "method": event.get("httpMethod")
    })

    # Log with append_keys for persistent keys
    logger.append_keys(user_id="123456")
    logger.info("User context added")

    # Log different levels
    logger.debug("Debug information")
    logger.warning("Warning message")

    try:
        # Business logic
        result = process_event(event)
        logger.info("Request processed successfully", extra={"result_status": "success"})
        return result
    except Exception as e:
        logger.exception("Error processing request")
        raise
```

### Custom Metrics

```python
# src/handlers/metrics_example.py
from aws_lambda_powertools import Metrics
from aws_lambda_powertools.metrics import MetricUnit

# Initialize metrics
metrics = Metrics(namespace="MyApplication")

@metrics.log_metrics
def lambda_handler(event, context):
    # Add default dimensions
    metrics.add_dimension(name="environment", value="prod")

    # Add metrics
    metrics.add_metric(name="ProcessedItems", unit=MetricUnit.Count, value=1)

    # Process items
    items = event.get('items', [])
    for item in items:
        process_item(item)
        # Add more metrics
        metrics.add_metric(name="ItemProcessingTime", unit=MetricUnit.Milliseconds, value=item.get('processing_time', 0))

    # Add metrics with custom dimensions
    metrics.add_dimension(name="function_version", value=context.function_version)
    metrics.add_metric(name="SuccessfulInvocations", unit=MetricUnit.Count, value=1)

    return {
        'statusCode': 200,
        'body': json.dumps({'processed': len(items)})
    }
```

## Common Patterns and Examples

### API Gateway Integration

```python
# src/handlers/api_gateway_handler.py
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """
    Handler for API Gateway events
    """
    logger.info(f"Received API Gateway event: {json.dumps(event)}")

    # Get path and method
    path = event.get('path', '')
    http_method = event.get('httpMethod', '')

    # Route the request
    if path == '/users' and http_method == 'GET':
        return get_users(event)
    elif path.startswith('/users/') and http_method == 'GET':
        user_id = path.split('/')[-1]
        return get_user(user_id, event)
    elif path == '/users' and http_method == 'POST':
        return create_user(event)
    else:
        return {
            'statusCode': 404,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({'error': 'Not Found'})
        }

def get_users(event):
    # Get query parameters
    query_params = event.get('queryStringParameters', {}) or {}
    limit = int(query_params.get('limit', 10))

    # Get users from database
    # ...

    return {
        'statusCode': 200,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps({'users': [{'id': '1', 'name': 'User 1'}, {'id': '2', 'name': 'User 2'}]})
    }

def get_user(user_id, event):
    # Get user from database
    # ...

    return {
        'statusCode': 200,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps({'id': user_id, 'name': f'User {user_id}'})
    }

def create_user(event):
    # Parse request body
    try:
        body = json.loads(event.get('body', '{}'))
    except json.JSONDecodeError:
        return {
            'statusCode': 400,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({'error': 'Invalid JSON in request body'})
        }

    # Validate request
    if 'name' not in body:
        return {
            'statusCode': 400,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({'error': 'Missing required field: name'})
        }

    # Create user in database
    # ...

    return {
        'statusCode': 201,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps({'id': '3', 'name': body['name']})
    }
```

### DynamoDB Integration

```python
# src/services/dynamodb_service.py
import boto3
from botocore.exceptions import ClientError
import logging

logger = logging.getLogger()

class DynamoDBService:
    def __init__(self, table_name):
        self.dynamodb = boto3.resource('dynamodb')
        self.table = self.dynamodb.Table(table_name)

    def get_item(self, key_name, key_value):
        """
        Get an item from DynamoDB
        """
        try:
            response = self.table.get_item(
                Key={key_name: key_value}
            )
            return response.get('Item')
        except ClientError as e:
            logger.error(f"Error getting item from DynamoDB: {str(e)}")
            raise

    def save_item(self, item):
        """
        Save an item to DynamoDB
        """
        try:
            response = self.table.put_item(
                Item=item
            )
            return response
        except ClientError as e:
            logger.error(f"Error saving item to DynamoDB: {str(e)}")
            raise

    def query_items(self, key_name, key_value, index_name=None):
        """
        Query items from DynamoDB
        """
        try:
            params = {
                'KeyConditionExpression': f"{key_name} = :value",
                'ExpressionAttributeValues': {
                    ':value': key_value
                }
            }

            if index_name:
                params['IndexName'] = index_name

            response = self.table.query(**params)
            return response.get('Items', [])
        except ClientError as e:
            logger.error(f"Error querying items from DynamoDB: {str(e)}")
            raise
```

### SQS Integration

```python
# src/handlers/sqs_handler.py
import json
import logging
from aws_lambda_powertools import Logger

logger = Logger()

@logger.inject_lambda_context
def lambda_handler(event, context):
    """
    Handler for SQS events
    """
    logger.info(f"Received SQS event with {len(event['Records'])} records")

    processed_records = 0
    failed_records = 0

    for record in event['Records']:
        try:
            # Extract message body
            message_body = record['body']
            logger.info(f"Processing message", extra={"messageId": record['messageId']})

            # Parse JSON if needed
            try:
                message_data = json.loads(message_body)
            except json.JSONDecodeError:
                logger.warning(f"Message is not valid JSON", extra={"messageId": record['messageId']})
                message_data = message_body

            # Process the message
            process_message(message_data)

            processed_records += 1
            logger.info(f"Successfully processed message", extra={"messageId": record['messageId']})
        except Exception as e:
            failed_records += 1
            logger.exception(f"Error processing message", extra={"messageId": record['messageId']})

    logger.info(f"Completed processing SQS batch", extra={
        "processed_records": processed_records,
        "failed_records": failed_records
    })

    return {
        "processed_records": processed_records,
        "failed_records": failed_records
    }

def process_message(message_data):
    """
    Process a message from SQS
    """
    # Implement your business logic here
    pass
```

### S3 Event Processing

```python
# src/handlers/s3_handler.py
import json
import urllib.parse
import boto3
from aws_lambda_powertools import Logger

logger = Logger()
s3_client = boto3.client('s3')

@logger.inject_lambda_context
def lambda_handler(event, context):
    """
    Handler for S3 events
    """
    logger.info(f"Received S3 event with {len(event['Records'])} records")

    for record in event['Records']:
        # Get bucket and key
        bucket = record['s3']['bucket']['name']
        key = urllib.parse.unquote_plus(record['s3']['object']['key'])

        logger.info(f"Processing S3 object", extra={"bucket": bucket, "key": key})

        try:
            # Get the object
            response = s3_client.get_object(Bucket=bucket, Key=key)
            content = response['Body'].read().decode('utf-8')

            # Process the content
            process_s3_object(bucket, key, content)

            logger.info(f"Successfully processed S3 object", extra={"bucket": bucket, "key": key})
        except Exception as e:
            logger.exception(f"Error processing S3 object", extra={"bucket": bucket, "key": key})
            raise

    return {
        "statusCode": 200,
        "body": json.dumps({"message": "Successfully processed S3 event"})
    }

def process_s3_object(bucket, key, content):
    """
    Process an S3 object
    """
    # Implement your business logic here
    pass
```

This document provides guidelines for developing and deploying Python Lambda functions using Terraform. For more information on general Lambda best practices, refer to the [Lambda Guidelines](lambda.md) document.
