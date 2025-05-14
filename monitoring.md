# Monitoring and Observability Best Practices for AWS API Gateway and Lambda with Terraform

This document outlines the best practices for implementing monitoring and observability for AWS API Gateway and Lambda functions using Terraform. Following these guidelines will help ensure your serverless applications are properly monitored, allowing you to detect and resolve issues quickly.

## Table of Contents

- [CloudWatch Logs Configuration](#cloudwatch-logs-configuration)
- [CloudWatch Metrics and Alarms](#cloudwatch-metrics-and-alarms)
- [X-Ray Tracing](#x-ray-tracing)
- [CloudWatch Dashboards](#cloudwatch-dashboards)
- [Centralized Logging](#centralized-logging)
- [Alerting and Notifications](#alerting-and-notifications)
- [Custom Metrics](#custom-metrics)
- [Cost Optimization](#cost-optimization)
- [Operational Readiness](#operational-readiness)

## CloudWatch Logs Configuration

### Lambda Logs

```hcl
# Create CloudWatch Log Group with appropriate retention
resource "aws_cloudwatch_log_group" "lambda_logs" {
  name              = "/aws/lambda/${aws_lambda_function.example.function_name}"
  retention_in_days = var.environment == "prod" ? 90 : 30

  # Enable encryption with KMS (recommended for production)
  kms_key_id = var.environment == "prod" ? aws_kms_key.logs_key[0].arn : null

  tags = {
    Environment = var.environment
    Project     = var.project_name
    Function    = aws_lambda_function.example.function_name
  }
}

# Reference the log group in the Lambda function
resource "aws_lambda_function" "example" {
  # ... other configuration ...

  # Explicitly depend on the log group to ensure it exists before the function
  depends_on = [aws_cloudwatch_log_group.lambda_logs]
}
```

### API Gateway Logs

```hcl
# Create CloudWatch Log Group for API Gateway
resource "aws_cloudwatch_log_group" "api_gateway_logs" {
  name              = "/aws/apigateway/${aws_api_gateway_rest_api.example.name}"
  retention_in_days = var.environment == "prod" ? 90 : 30

  tags = {
    Environment = var.environment
    Project     = var.project_name
    API         = aws_api_gateway_rest_api.example.name
  }
}

# Configure API Gateway stage with logging
resource "aws_api_gateway_stage" "example" {
  # ... other configuration ...

  # Enable CloudWatch logs
  access_log_settings {
    destination_arn = aws_cloudwatch_log_group.api_gateway_logs.arn
    format = jsonencode({
      requestId               = "$context.requestId"
      ip                      = "$context.identity.sourceIp"
      caller                  = "$context.identity.caller"
      user                    = "$context.identity.user"
      requestTime             = "$context.requestTime"
      httpMethod              = "$context.httpMethod"
      resourcePath            = "$context.resourcePath"
      status                  = "$context.status"
      protocol                = "$context.protocol"
      responseLength          = "$context.responseLength"
      integrationErrorMessage = "$context.integrationErrorMessage"
      integrationLatency      = "$context.integrationLatency"
      responseLatency         = "$context.responseLatency"
    })
  }
}

# IAM role for API Gateway to write logs to CloudWatch
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

resource "aws_iam_role_policy" "api_gateway_cloudwatch" {
  name = "cloudwatch-logs-policy"
  role = aws_iam_role.api_gateway_cloudwatch.id

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
      Resource = "*"
    }]
  })
}

# Set up account-level settings for API Gateway
resource "aws_api_gateway_account" "main" {
  cloudwatch_role_arn = aws_iam_role.api_gateway_cloudwatch.arn
}
```

### Log Insights Queries

Store commonly used CloudWatch Logs Insights queries as outputs:

```hcl
output "cloudwatch_logs_insights_queries" {
  description = "Useful CloudWatch Logs Insights queries for troubleshooting"
  value = {
    lambda_errors = {
      log_group = aws_cloudwatch_log_group.lambda_logs.name
      query     = "fields @timestamp, @message | filter @message like /ERROR/ or @message like /Exception/ | sort @timestamp desc | limit 20"
    }
    lambda_cold_starts = {
      log_group = aws_cloudwatch_log_group.lambda_logs.name
      query     = "filter @type = \"REPORT\" | fields @requestId, @duration, @initDuration | sort @initDuration desc | limit 20"
    }
    api_gateway_4xx_errors = {
      log_group = aws_cloudwatch_log_group.api_gateway_logs.name
      query     = "fields @timestamp, status, httpMethod, resourcePath | filter status >= 400 and status < 500 | sort @timestamp desc | limit 20"
    }
    api_gateway_5xx_errors = {
      log_group = aws_cloudwatch_log_group.api_gateway_logs.name
      query     = "fields @timestamp, status, httpMethod, resourcePath, integrationErrorMessage | filter status >= 500 | sort @timestamp desc | limit 20"
    }
    api_gateway_slow_responses = {
      log_group = aws_cloudwatch_log_group.api_gateway_logs.name
      query     = "fields @timestamp, responseLatency, httpMethod, resourcePath | filter responseLatency > 1000 | sort responseLatency desc | limit 20"
    }
  }
}
```

## CloudWatch Metrics and Alarms

### Lambda Alarms

```hcl
# SNS Topic for alarms
resource "aws_sns_topic" "alarms" {
  name = "${var.project_name}-${var.environment}-alarms"
}

# Lambda Error Alarm
resource "aws_cloudwatch_metric_alarm" "lambda_errors" {
  alarm_name          = "${aws_lambda_function.example.function_name}-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  period              = 60
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "This metric monitors lambda function errors"
  treat_missing_data  = "notBreaching"

  dimensions = {
    FunctionName = aws_lambda_function.example.function_name
  }

  alarm_actions = [aws_sns_topic.alarms.arn]
  ok_actions    = [aws_sns_topic.alarms.arn]
}

# Lambda Throttles Alarm
resource "aws_cloudwatch_metric_alarm" "lambda_throttles" {
  alarm_name          = "${aws_lambda_function.example.function_name}-throttles"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "Throttles"
  namespace           = "AWS/Lambda"
  period              = 60
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "This metric monitors lambda function throttles"
  treat_missing_data  = "notBreaching"

  dimensions = {
    FunctionName = aws_lambda_function.example.function_name
  }

  alarm_actions = [aws_sns_topic.alarms.arn]
  ok_actions    = [aws_sns_topic.alarms.arn]
}

# Lambda Duration Alarm
resource "aws_cloudwatch_metric_alarm" "lambda_duration" {
  alarm_name          = "${aws_lambda_function.example.function_name}-duration"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "Duration"
  namespace           = "AWS/Lambda"
  period              = 60
  statistic           = "Average"
  threshold           = aws_lambda_function.example.timeout * 1000 * 0.8  # 80% of timeout
  alarm_description   = "This metric monitors lambda function duration approaching timeout"
  treat_missing_data  = "notBreaching"

  dimensions = {
    FunctionName = aws_lambda_function.example.function_name
  }

  alarm_actions = [aws_sns_topic.alarms.arn]
  ok_actions    = [aws_sns_topic.alarms.arn]
}

# Lambda Invocations Alarm (for detecting unusual traffic patterns)
resource "aws_cloudwatch_metric_alarm" "lambda_invocations_high" {
  alarm_name          = "${aws_lambda_function.example.function_name}-high-invocations"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "Invocations"
  namespace           = "AWS/Lambda"
  period              = 60
  statistic           = "Sum"
  threshold           = var.lambda_invocations_threshold
  alarm_description   = "This metric monitors unusually high lambda function invocations"
  treat_missing_data  = "notBreaching"

  dimensions = {
    FunctionName = aws_lambda_function.example.function_name
  }

  alarm_actions = [aws_sns_topic.alarms.arn]
  ok_actions    = [aws_sns_topic.alarms.arn]
}
```

### API Gateway Alarms

```hcl
# API Gateway 4XX Errors Alarm
resource "aws_cloudwatch_metric_alarm" "api_4xx_errors" {
  alarm_name          = "${aws_api_gateway_rest_api.example.name}-4xx-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "4XXError"
  namespace           = "AWS/ApiGateway"
  period              = 60
  statistic           = "Sum"
  threshold           = var.environment == "prod" ? 10 : 50
  alarm_description   = "This metric monitors API Gateway 4XX errors"
  treat_missing_data  = "notBreaching"

  dimensions = {
    ApiName  = aws_api_gateway_rest_api.example.name
    Stage    = aws_api_gateway_stage.example.stage_name
  }

  alarm_actions = [aws_sns_topic.alarms.arn]
  ok_actions    = [aws_sns_topic.alarms.arn]
}

# API Gateway 5XX Errors Alarm
resource "aws_cloudwatch_metric_alarm" "api_5xx_errors" {
  alarm_name          = "${aws_api_gateway_rest_api.example.name}-5xx-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "5XXError"
  namespace           = "AWS/ApiGateway"
  period              = 60
  statistic           = "Sum"
  threshold           = var.environment == "prod" ? 5 : 20
  alarm_description   = "This metric monitors API Gateway 5XX errors"
  treat_missing_data  = "notBreaching"

  dimensions = {
    ApiName  = aws_api_gateway_rest_api.example.name
    Stage    = aws_api_gateway_stage.example.stage_name
  }

  alarm_actions = [aws_sns_topic.alarms.arn]
  ok_actions    = [aws_sns_topic.alarms.arn]
}

# API Gateway Latency Alarm
resource "aws_cloudwatch_metric_alarm" "api_latency" {
  alarm_name          = "${aws_api_gateway_rest_api.example.name}-latency"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "Latency"
  namespace           = "AWS/ApiGateway"
  period              = 60
  statistic           = "p90"
  threshold           = var.environment == "prod" ? 1000 : 2000  # 1 second for prod, 2 seconds for non-prod
  alarm_description   = "This metric monitors API Gateway latency"
  treat_missing_data  = "notBreaching"

  dimensions = {
    ApiName  = aws_api_gateway_rest_api.example.name
    Stage    = aws_api_gateway_stage.example.stage_name
  }

  alarm_actions = [aws_sns_topic.alarms.arn]
  ok_actions    = [aws_sns_topic.alarms.arn]
}

# API Gateway Integration Latency Alarm
resource "aws_cloudwatch_metric_alarm" "api_integration_latency" {
  alarm_name          = "${aws_api_gateway_rest_api.example.name}-integration-latency"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "IntegrationLatency"
  namespace           = "AWS/ApiGateway"
  period              = 60
  statistic           = "p90"
  threshold           = var.environment == "prod" ? 800 : 1500  # 800ms for prod, 1.5 seconds for non-prod
  alarm_description   = "This metric monitors API Gateway integration latency"
  treat_missing_data  = "notBreaching"

  dimensions = {
    ApiName  = aws_api_gateway_rest_api.example.name
    Stage    = aws_api_gateway_stage.example.stage_name
  }

  alarm_actions = [aws_sns_topic.alarms.arn]
  ok_actions    = [aws_sns_topic.alarms.arn]
}
```

### Composite Alarms

```hcl
# Create a composite alarm for critical API issues
resource "aws_cloudwatch_composite_alarm" "api_critical" {
  alarm_name        = "${var.project_name}-${var.environment}-api-critical"
  alarm_description = "Composite alarm for critical API issues"

  alarm_rule = "ALARM(${aws_cloudwatch_metric_alarm.api_5xx_errors.alarm_name}) OR ALARM(${aws_cloudwatch_metric_alarm.lambda_errors.alarm_name})"

  alarm_actions = [aws_sns_topic.critical_alarms.arn]
  ok_actions    = [aws_sns_topic.critical_alarms.arn]
}
```

## X-Ray Tracing

### Enable X-Ray for Lambda

```hcl
# IAM policy for X-Ray
resource "aws_iam_policy" "lambda_xray" {
  name        = "${var.project_name}-${var.environment}-lambda-xray"
  description = "IAM policy for Lambda to use X-Ray tracing"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = [
        "xray:PutTraceSegments",
        "xray:PutTelemetryRecords",
        "xray:GetSamplingRules",
        "xray:GetSamplingTargets",
        "xray:GetSamplingStatisticSummaries"
      ]
      Effect   = "Allow"
      Resource = "*"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_xray" {
  role       = aws_iam_role.lambda_execution.name
  policy_arn = aws_iam_policy.lambda_xray.arn
}

# Enable X-Ray tracing for Lambda
resource "aws_lambda_function" "example" {
  # ... other configuration ...

  tracing_config {
    mode = "Active"  # Options: PassThrough, Active
  }
}
```

### Enable X-Ray for API Gateway

```hcl
# Enable X-Ray tracing for API Gateway stage
resource "aws_api_gateway_stage" "example" {
  # ... other configuration ...

  xray_tracing_enabled = true
}
```

### X-Ray Sampling Rule

```hcl
resource "aws_xray_sampling_rule" "example" {
  rule_name      = "${var.project_name}-${var.environment}-sampling"
  priority       = 1000
  reservoir_size = 1
  fixed_rate     = var.environment == "prod" ? 0.05 : 0.10  # Sample 5% in prod, 10% in non-prod
  url_path       = "*"
  host           = "*"
  http_method    = "*"
  service_name   = var.project_name
  service_type   = "*"

  attributes = {
    Environment = var.environment
  }
}
```

## CloudWatch Dashboards

### Comprehensive Dashboard

```hcl
resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = "${var.project_name}-${var.environment}-dashboard"

  dashboard_body = jsonencode({
    widgets = [
      # API Gateway Overview
      {
        type   = "text",
        x      = 0,
        y      = 0,
        width  = 24,
        height = 1,
        properties = {
          markdown = "# API Gateway Metrics - ${var.environment} Environment"
        }
      },
      # API Gateway Requests
      {
        type   = "metric",
        x      = 0,
        y      = 1,
        width  = 8,
        height = 6,
        properties = {
          metrics = [
            ["AWS/ApiGateway", "Count", "ApiName", aws_api_gateway_rest_api.example.name, "Stage", aws_api_gateway_stage.example.stage_name]
          ],
          view    = "timeSeries",
          stacked = false,
          region  = var.aws_region,
          title   = "API Requests",
          period  = 60
        }
      },
      # API Gateway Latency
      {
        type   = "metric",
        x      = 8,
        y      = 1,
        width  = 8,
        height = 6,
        properties = {
          metrics = [
            ["AWS/ApiGateway", "Latency", "ApiName", aws_api_gateway_rest_api.example.name, "Stage", aws_api_gateway_stage.example.stage_name, { stat = "p50" }],
            ["AWS/ApiGateway", "Latency", "ApiName", aws_api_gateway_rest_api.example.name, "Stage", aws_api_gateway_stage.example.stage_name, { stat = "p90" }],
            ["AWS/ApiGateway", "Latency", "ApiName", aws_api_gateway_rest_api.example.name, "Stage", aws_api_gateway_stage.example.stage_name, { stat = "p99" }]
          ],
          view    = "timeSeries",
          stacked = false,
          region  = var.aws_region,
          title   = "API Latency",
          period  = 60
        }
      },
      # API Gateway Errors
      {
        type   = "metric",
        x      = 16,
        y      = 1,
        width  = 8,
        height = 6,
        properties = {
          metrics = [
            ["AWS/ApiGateway", "4XXError", "ApiName", aws_api_gateway_rest_api.example.name, "Stage", aws_api_gateway_stage.example.stage_name],
            ["AWS/ApiGateway", "5XXError", "ApiName", aws_api_gateway_rest_api.example.name, "Stage", aws_api_gateway_stage.example.stage_name]
          ],
          view    = "timeSeries",
          stacked = false,
          region  = var.aws_region,
          title   = "API Errors",
          period  = 60
        }
      },
      # Lambda Overview
      {
        type   = "text",
        x      = 0,
        y      = 7,
        width  = 24,
        height = 1,
        properties = {
          markdown = "# Lambda Metrics - ${var.environment} Environment"
        }
      },
      # Lambda Invocations
      {
        type   = "metric",
        x      = 0,
        y      = 8,
        width  = 8,
        height = 6,
        properties = {
          metrics = [
            ["AWS/Lambda", "Invocations", "FunctionName", aws_lambda_function.example.function_name]
          ],
          view    = "timeSeries",
          stacked = false,
          region  = var.aws_region,
          title   = "Lambda Invocations",
          period  = 60
        }
      },
      # Lambda Duration
      {
        type   = "metric",
        x      = 8,
        y      = 8,
        width  = 8,
        height = 6,
        properties = {
          metrics = [
            ["AWS/Lambda", "Duration", "FunctionName", aws_lambda_function.example.function_name, { stat = "p50" }],
            ["AWS/Lambda", "Duration", "FunctionName", aws_lambda_function.example.function_name, { stat = "p90" }],
            ["AWS/Lambda", "Duration", "FunctionName", aws_lambda_function.example.function_name, { stat = "p99" }]
          ],
          view    = "timeSeries",
          stacked = false,
          region  = var.aws_region,
          title   = "Lambda Duration",
          period  = 60
        }
      },
      # Lambda Errors and Throttles
      {
        type   = "metric",
        x      = 16,
        y      = 8,
        width  = 8,
        height = 6,
        properties = {
          metrics = [
            ["AWS/Lambda", "Errors", "FunctionName", aws_lambda_function.example.function_name],
            ["AWS/Lambda", "Throttles", "FunctionName", aws_lambda_function.example.function_name]
          ],
          view    = "timeSeries",
          stacked = false,
          region  = var.aws_region,
          title   = "Lambda Errors and Throttles",
          period  = 60
        }
      },
      # Lambda Concurrent Executions
      {
        type   = "metric",
        x      = 0,
        y      = 14,
        width  = 8,
        height = 6,
        properties = {
          metrics = [
            ["AWS/Lambda", "ConcurrentExecutions", "FunctionName", aws_lambda_function.example.function_name]
          ],
          view    = "timeSeries",
          stacked = false,
          region  = var.aws_region,
          title   = "Lambda Concurrent Executions",
          period  = 60
        }
      },
      # Lambda Iterative Age
      {
        type   = "metric",
        x      = 8,
        y      = 14,
        width  = 8,
        height = 6,
        properties = {
          metrics = [
            ["AWS/Lambda", "IteratorAge", "FunctionName", aws_lambda_function.example.function_name]
          ],
          view    = "timeSeries",
          stacked = false,
          region  = var.aws_region,
          title   = "Lambda Iterator Age (for event source mappings)",
          period  = 60
        }
      },
      # Lambda Cold Starts (Custom Metric)
      {
        type   = "log",
        x      = 16,
        y      = 14,
        width  = 8,
        height = 6,
        properties = {
          query   = "SOURCE '/aws/lambda/${aws_lambda_function.example.function_name}' | filter @type = \"REPORT\" | filter @initDuration > 0 | stats count() as coldStarts by bin(5m)",
          region  = var.aws_region,
          title   = "Lambda Cold Starts",
          view    = "timeSeries"
        }
      }
    ]
  })
}
```

### Per-Endpoint Dashboard

```hcl
# Create a dashboard for each API endpoint
resource "aws_cloudwatch_dashboard" "api_endpoints" {
  for_each = var.api_endpoints

  dashboard_name = "${var.project_name}-${var.environment}-${each.key}-dashboard"

  dashboard_body = jsonencode({
    widgets = [
      # Endpoint Overview
      {
        type   = "text",
        x      = 0,
        y      = 0,
        width  = 24,
        height = 1,
        properties = {
          markdown = "# API Endpoint: ${each.key} - ${var.environment} Environment"
        }
      },
      # Endpoint Requests
      {
        type   = "metric",
        x      = 0,
        y      = 1,
        width  = 8,
        height = 6,
        properties = {
          metrics = [
            ["AWS/ApiGateway", "Count", "ApiName", aws_api_gateway_rest_api.example.name, "Stage", aws_api_gateway_stage.example.stage_name, "Resource", each.value.resource, "Method", each.value.method]
          ],
          view    = "timeSeries",
          stacked = false,
          region  = var.aws_region,
          title   = "Requests",
          period  = 60
        }
      },
      # Endpoint Latency
      {
        type   = "metric",
        x      = 8,
        y      = 1,
        width  = 8,
        height = 6,
        properties = {
          metrics = [
            ["AWS/ApiGateway", "Latency", "ApiName", aws_api_gateway_rest_api.example.name, "Stage", aws_api_gateway_stage.example.stage_name, "Resource", each.value.resource, "Method", each.value.method, { stat = "p50" }],
            ["AWS/ApiGateway", "Latency", "ApiName", aws_api_gateway_rest_api.example.name, "Stage", aws_api_gateway_stage.example.stage_name, "Resource", each.value.resource, "Method", each.value.method, { stat = "p90" }],
            ["AWS/ApiGateway", "Latency", "ApiName", aws_api_gateway_rest_api.example.name, "Stage", aws_api_gateway_stage.example.stage_name, "Resource", each.value.resource, "Method", each.value.method, { stat = "p99" }]
          ],
          view    = "timeSeries",
          stacked = false,
          region  = var.aws_region,
          title   = "Latency",
          period  = 60
        }
      },
      # Endpoint Errors
      {
        type   = "metric",
        x      = 16,
        y      = 1,
        width  = 8,
        height = 6,
        properties = {
          metrics = [
            ["AWS/ApiGateway", "4XXError", "ApiName", aws_api_gateway_rest_api.example.name, "Stage", aws_api_gateway_stage.example.stage_name, "Resource", each.value.resource, "Method", each.value.method],
            ["AWS/ApiGateway", "5XXError", "ApiName", aws_api_gateway_rest_api.example.name, "Stage", aws_api_gateway_stage.example.stage_name, "Resource", each.value.resource, "Method", each.value.method]
          ],
          view    = "timeSeries",
          stacked = false,
          region  = var.aws_region,
          title   = "Errors",
          period  = 60
        }
      },
      # Lambda Function for this endpoint
      {
        type   = "metric",
        x      = 0,
        y      = 7,
        width  = 8,
        height = 6,
        properties = {
          metrics = [
            ["AWS/Lambda", "Invocations", "FunctionName", each.value.lambda_function]
          ],
          view    = "timeSeries",
          stacked = false,
          region  = var.aws_region,
          title   = "Lambda Invocations",
          period  = 60
        }
      },
      # Lambda Duration
      {
        type   = "metric",
        x      = 8,
        y      = 7,
        width  = 8,
        height = 6,
        properties = {
          metrics = [
            ["AWS/Lambda", "Duration", "FunctionName", each.value.lambda_function, { stat = "p50" }],
            ["AWS/Lambda", "Duration", "FunctionName", each.value.lambda_function, { stat = "p90" }],
            ["AWS/Lambda", "Duration", "FunctionName", each.value.lambda_function, { stat = "p99" }]
          ],
          view    = "timeSeries",
          stacked = false,
          region  = var.aws_region,
          title   = "Lambda Duration",
          period  = 60
        }
      },
      # Lambda Errors
      {
        type   = "metric",
        x      = 16,
        y      = 7,
        width  = 8,
        height = 6,
        properties = {
          metrics = [
            ["AWS/Lambda", "Errors", "FunctionName", each.value.lambda_function]
          ],
          view    = "timeSeries",
          stacked = false,
          region  = var.aws_region,
          title   = "Lambda Errors",
          period  = 60
        }
      }
    ]
  })
}
```

## Centralized Logging

### CloudWatch Logs Subscription Filter

```hcl
# Create a Kinesis Data Firehose delivery stream to S3
resource "aws_kinesis_firehose_delivery_stream" "logs_to_s3" {
  name        = "${var.project_name}-${var.environment}-logs-to-s3"
  destination = "s3"

  s3_configuration {
    role_arn   = aws_iam_role.firehose_role.arn
    bucket_arn = aws_s3_bucket.logs_bucket.arn
    prefix     = "logs/${var.environment}/"

    # Buffer settings
    buffer_size     = 5  # MB
    buffer_interval = 300  # seconds

    # Compression
    compression_format = "GZIP"
  }

  tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}

# IAM role for Firehose
resource "aws_iam_role" "firehose_role" {
  name = "${var.project_name}-${var.environment}-firehose-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "firehose.amazonaws.com"
      }
    }]
  })
}

# IAM policy for Firehose to write to S3
resource "aws_iam_policy" "firehose_s3" {
  name        = "${var.project_name}-${var.environment}-firehose-s3"
  description = "IAM policy for Firehose to write to S3"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = [
        "s3:AbortMultipartUpload",
        "s3:GetBucketLocation",
        "s3:GetObject",
        "s3:ListBucket",
        "s3:ListBucketMultipartUploads",
        "s3:PutObject"
      ]
      Effect   = "Allow"
      Resource = [
        aws_s3_bucket.logs_bucket.arn,
        "${aws_s3_bucket.logs_bucket.arn}/*"
      ]
    }]
  })
}

resource "aws_iam_role_policy_attachment" "firehose_s3" {
  role       = aws_iam_role.firehose_role.name
  policy_arn = aws_iam_policy.firehose_s3.arn
}

# IAM role for CloudWatch Logs to send to Firehose
resource "aws_iam_role" "cloudwatch_to_firehose" {
  name = "${var.project_name}-${var.environment}-cloudwatch-to-firehose"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "logs.amazonaws.com"
      }
    }]
  })
}

# IAM policy for CloudWatch Logs to put records to Firehose
resource "aws_iam_policy" "cloudwatch_to_firehose" {
  name        = "${var.project_name}-${var.environment}-cloudwatch-to-firehose"
  description = "IAM policy for CloudWatch Logs to put records to Firehose"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = [
        "firehose:PutRecord",
        "firehose:PutRecordBatch"
      ]
      Effect   = "Allow"
      Resource = aws_kinesis_firehose_delivery_stream.logs_to_s3.arn
    }]
  })
}

resource "aws_iam_role_policy_attachment" "cloudwatch_to_firehose" {
  role       = aws_iam_role.cloudwatch_to_firehose.name
  policy_arn = aws_iam_policy.cloudwatch_to_firehose.arn
}

# Subscription filter for Lambda logs
resource "aws_cloudwatch_log_subscription_filter" "lambda_logs_to_firehose" {
  name            = "${var.project_name}-${var.environment}-lambda-logs-to-firehose"
  log_group_name  = aws_cloudwatch_log_group.lambda_logs.name
  destination_arn = aws_kinesis_firehose_delivery_stream.logs_to_s3.arn
  filter_pattern  = ""  # Empty pattern means all logs
  role_arn        = aws_iam_role.cloudwatch_to_firehose.arn
}

# Subscription filter for API Gateway logs
resource "aws_cloudwatch_log_subscription_filter" "api_gateway_logs_to_firehose" {
  name            = "${var.project_name}-${var.environment}-api-logs-to-firehose"
  log_group_name  = aws_cloudwatch_log_group.api_gateway_logs.name
  destination_arn = aws_kinesis_firehose_delivery_stream.logs_to_s3.arn
  filter_pattern  = ""  # Empty pattern means all logs
  role_arn        = aws_iam_role.cloudwatch_to_firehose.arn
}
```

### OpenSearch Integration

```hcl
# Create OpenSearch domain
resource "aws_opensearch_domain" "logs" {
  domain_name    = "${var.project_name}-${var.environment}-logs"
  engine_version = "OpenSearch_2.5"

  cluster_config {
    instance_type            = var.environment == "prod" ? "m5.large.search" : "t3.small.search"
    instance_count           = var.environment == "prod" ? 3 : 1
    zone_awareness_enabled   = var.environment == "prod" ? true : false

    zone_awareness_config {
      availability_zone_count = var.environment == "prod" ? 3 : 1
    }
  }

  ebs_options {
    ebs_enabled = true
    volume_size = var.environment == "prod" ? 100 : 10
    volume_type = "gp2"
  }

  encrypt_at_rest {
    enabled = true
  }

  node_to_node_encryption {
    enabled = true
  }

  domain_endpoint_options {
    enforce_https       = true
    tls_security_policy = "Policy-Min-TLS-1-2-2019-07"
  }

  advanced_security_options {
    enabled                        = true
    internal_user_database_enabled = true

    master_user_options {
      master_user_name     = var.opensearch_master_user
      master_user_password = var.opensearch_master_password
    }
  }

  tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}

# Configure Firehose to deliver to OpenSearch
resource "aws_kinesis_firehose_delivery_stream" "logs_to_opensearch" {
  name        = "${var.project_name}-${var.environment}-logs-to-opensearch"
  destination = "elasticsearch"

  elasticsearch_configuration {
    domain_arn            = aws_opensearch_domain.logs.arn
    role_arn              = aws_iam_role.firehose_opensearch_role.arn
    index_name            = "logs"
    index_rotation_period = "OneDay"

    # Buffer settings
    buffering_interval = 60
    buffering_size     = 5

    # Retry settings
    retry_duration = 300

    # S3 backup for failed documents
    s3_backup_mode = "FailedDocumentsOnly"

    # CloudWatch logging for delivery errors
    cloudwatch_logging_options {
      enabled         = true
      log_group_name  = aws_cloudwatch_log_group.firehose_logs.name
      log_stream_name = "ElasticsearchDelivery"
    }
  }

  s3_configuration {
    role_arn   = aws_iam_role.firehose_opensearch_role.arn
    bucket_arn = aws_s3_bucket.logs_backup_bucket.arn
    prefix     = "opensearch-failed/"

    # Buffer settings
    buffer_size     = 5
    buffer_interval = 300

    # Compression
    compression_format = "GZIP"
  }

  tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}
```

## Alerting and Notifications

### SNS Topics and Subscriptions

```hcl
# Create different SNS topics for different alert severities
resource "aws_sns_topic" "critical_alerts" {
  name = "${var.project_name}-${var.environment}-critical-alerts"

  # Enable server-side encryption
  kms_master_key_id = aws_kms_key.sns.id
}

resource "aws_sns_topic" "warning_alerts" {
  name = "${var.project_name}-${var.environment}-warning-alerts"

  # Enable server-side encryption
  kms_master_key_id = aws_kms_key.sns.id
}

resource "aws_sns_topic" "info_alerts" {
  name = "${var.project_name}-${var.environment}-info-alerts"

  # Enable server-side encryption
  kms_master_key_id = aws_kms_key.sns.id
}

# Email subscriptions
resource "aws_sns_topic_subscription" "critical_email" {
  topic_arn = aws_sns_topic.critical_alerts.arn
  protocol  = "email"
  endpoint  = var.critical_alerts_email
}

resource "aws_sns_topic_subscription" "warning_email" {
  topic_arn = aws_sns_topic.warning_alerts.arn
  protocol  = "email"
  endpoint  = var.warning_alerts_email
}

# SMS subscriptions for critical alerts
resource "aws_sns_topic_subscription" "critical_sms" {
  count     = var.environment == "prod" ? 1 : 0
  topic_arn = aws_sns_topic.critical_alerts.arn
  protocol  = "sms"
  endpoint  = var.on_call_phone_number
}

# Slack integration via Lambda
resource "aws_lambda_function" "slack_notification" {
  function_name    = "${var.project_name}-${var.environment}-slack-notification"
  filename         = "slack_notification_lambda.zip"
  source_code_hash = filebase64sha256("slack_notification_lambda.zip")
  handler          = "index.handler"
  runtime          = "nodejs18.x"
  role             = aws_iam_role.lambda_slack_role.arn

  environment {
    variables = {
      SLACK_WEBHOOK_URL = var.slack_webhook_url
      SLACK_CHANNEL     = var.slack_channel
    }
  }

  tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}

resource "aws_sns_topic_subscription" "critical_slack" {
  topic_arn = aws_sns_topic.critical_alerts.arn
  protocol  = "lambda"
  endpoint  = aws_lambda_function.slack_notification.arn
}

resource "aws_sns_topic_subscription" "warning_slack" {
  topic_arn = aws_sns_topic.warning_alerts.arn
  protocol  = "lambda"
  endpoint  = aws_lambda_function.slack_notification.arn
}

# Lambda permission to allow SNS to invoke the function
resource "aws_lambda_permission" "sns_slack" {
  statement_id  = "AllowSNSInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.slack_notification.function_name
  principal     = "sns.amazonaws.com"
  source_arn    = aws_sns_topic.critical_alerts.arn
}
```

### CloudWatch Alarm Actions

```hcl
# Update the Lambda error alarm to use the critical alerts topic
resource "aws_cloudwatch_metric_alarm" "lambda_errors" {
  # ... existing configuration ...

  alarm_actions = [aws_sns_topic.critical_alerts.arn]
  ok_actions    = [aws_sns_topic.info_alerts.arn]
}

# Update the API Gateway 5XX error alarm to use the critical alerts topic
resource "aws_cloudwatch_metric_alarm" "api_5xx_errors" {
  # ... existing configuration ...

  alarm_actions = [aws_sns_topic.critical_alerts.arn]
  ok_actions    = [aws_sns_topic.info_alerts.arn]
}

# Update the API Gateway 4XX error alarm to use the warning alerts topic
resource "aws_cloudwatch_metric_alarm" "api_4xx_errors" {
  # ... existing configuration ...

  alarm_actions = [aws_sns_topic.warning_alerts.arn]
  ok_actions    = [aws_sns_topic.info_alerts.arn]
}
```

### EventBridge Rules for Notifications

```hcl
# EventBridge rule for Lambda deployment notifications
resource "aws_cloudwatch_event_rule" "lambda_deployment" {
  name        = "${var.project_name}-${var.environment}-lambda-deployment"
  description = "Capture Lambda function deployments"

  event_pattern = jsonencode({
    source      = ["aws.lambda"]
    detail-type = ["AWS API Call via CloudTrail"]
    detail = {
      eventSource = ["lambda.amazonaws.com"]
      eventName   = [
        "UpdateFunctionCode",
        "UpdateFunctionConfiguration",
        "PublishVersion"
      ]
    }
  })
}

resource "aws_cloudwatch_event_target" "lambda_deployment_notification" {
  rule      = aws_cloudwatch_event_rule.lambda_deployment.name
  target_id = "SendToSNS"
  arn       = aws_sns_topic.info_alerts.arn
}

# EventBridge rule for API Gateway deployment notifications
resource "aws_cloudwatch_event_rule" "api_gateway_deployment" {
  name        = "${var.project_name}-${var.environment}-api-gateway-deployment"
  description = "Capture API Gateway deployments"

  event_pattern = jsonencode({
    source      = ["aws.apigateway"]
    detail-type = ["AWS API Call via CloudTrail"]
    detail = {
      eventSource = ["apigateway.amazonaws.com"]
      eventName   = [
        "CreateDeployment",
        "UpdateStage"
      ]
    }
  })
}

resource "aws_cloudwatch_event_target" "api_gateway_deployment_notification" {
  rule      = aws_cloudwatch_event_rule.api_gateway_deployment.name
  target_id = "SendToSNS"
  arn       = aws_sns_topic.info_alerts.arn
}
```

## Custom Metrics

### Lambda Custom Metrics

```hcl
# Lambda function to publish custom metrics
resource "aws_lambda_function" "custom_metrics" {
  function_name    = "${var.project_name}-${var.environment}-custom-metrics"
  filename         = "custom_metrics_lambda.zip"
  source_code_hash = filebase64sha256("custom_metrics_lambda.zip")
  handler          = "index.handler"
  runtime          = "nodejs18.x"
  role             = aws_iam_role.lambda_metrics_role.arn

  environment {
    variables = {
      ENVIRONMENT = var.environment
      PROJECT     = var.project_name
    }
  }

  tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}

# IAM role for the custom metrics Lambda
resource "aws_iam_role" "lambda_metrics_role" {
  name = "${var.project_name}-${var.environment}-lambda-metrics-role"

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

# IAM policy for publishing CloudWatch metrics
resource "aws_iam_policy" "cloudwatch_put_metrics" {
  name        = "${var.project_name}-${var.environment}-cloudwatch-put-metrics"
  description = "Allow publishing CloudWatch metrics"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = [
        "cloudwatch:PutMetricData"
      ]
      Effect   = "Allow"
      Resource = "*"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_metrics_cloudwatch" {
  role       = aws_iam_role.lambda_metrics_role.name
  policy_arn = aws_iam_policy.cloudwatch_put_metrics.arn
}

# Basic logging permissions
resource "aws_iam_role_policy_attachment" "lambda_metrics_logs" {
  role       = aws_iam_role.lambda_metrics_role.name
  policy_arn = aws_iam_policy.lambda_logging.arn
}
```

### Example Custom Metrics Lambda Code

```javascript
// This would be in your custom_metrics_lambda.zip file
exports.handler = async (event) => {
  const AWS = require('aws-sdk');
  const cloudwatch = new AWS.CloudWatch();

  // Example: Calculate business metrics from your data
  const successfulTransactions = Math.floor(Math.random() * 100);
  const failedTransactions = Math.floor(Math.random() * 10);
  const totalTransactions = successfulTransactions + failedTransactions;
  const successRate = (successfulTransactions / totalTransactions) * 100;

  // Publish custom metrics
  await cloudwatch.putMetricData({
    Namespace: 'CustomBusinessMetrics',
    MetricData: [
      {
        MetricName: 'SuccessfulTransactions',
        Value: successfulTransactions,
        Unit: 'Count',
        Dimensions: [
          {
            Name: 'Environment',
            Value: process.env.ENVIRONMENT
          },
          {
            Name: 'Project',
            Value: process.env.PROJECT
          }
        ]
      },
      {
        MetricName: 'FailedTransactions',
        Value: failedTransactions,
        Unit: 'Count',
        Dimensions: [
          {
            Name: 'Environment',
            Value: process.env.ENVIRONMENT
          },
          {
            Name: 'Project',
            Value: process.env.PROJECT
          }
        ]
      },
      {
        MetricName: 'SuccessRate',
        Value: successRate,
        Unit: 'Percent',
        Dimensions: [
          {
            Name: 'Environment',
            Value: process.env.ENVIRONMENT
          },
          {
            Name: 'Project',
            Value: process.env.PROJECT
          }
        ]
      }
    ]
  }).promise();

  return {
    statusCode: 200,
    body: JSON.stringify('Custom metrics published successfully')
  };
};
```

### CloudWatch Alarms for Custom Metrics

```hcl
# Alarm for low success rate
resource "aws_cloudwatch_metric_alarm" "success_rate_low" {
  alarm_name          = "${var.project_name}-${var.environment}-success-rate-low"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 1
  metric_name         = "SuccessRate"
  namespace           = "CustomBusinessMetrics"
  period              = 60
  statistic           = "Average"
  threshold           = 95
  alarm_description   = "This metric monitors the success rate of transactions"
  treat_missing_data  = "notBreaching"

  dimensions = {
    Environment = var.environment
    Project     = var.project_name
  }

  alarm_actions = [aws_sns_topic.warning_alerts.arn]
  ok_actions    = [aws_sns_topic.info_alerts.arn]
}
```

## Cost Optimization

### Log Retention Policies

```hcl
# Set appropriate log retention periods
resource "aws_cloudwatch_log_group" "lambda_logs" {
  # ... existing configuration ...

  # Use shorter retention for non-prod environments
  retention_in_days = var.environment == "prod" ? 90 : 14
}

resource "aws_cloudwatch_log_group" "api_gateway_logs" {
  # ... existing configuration ...

  # Use shorter retention for non-prod environments
  retention_in_days = var.environment == "prod" ? 90 : 14
}
```

### Metric Filter Optimization

```hcl
# Create metric filters only for important log patterns
resource "aws_cloudwatch_log_metric_filter" "lambda_error" {
  name           = "${var.project_name}-${var.environment}-lambda-error-filter"
  pattern        = "ERROR"
  log_group_name = aws_cloudwatch_log_group.lambda_logs.name

  metric_transformation {
    name      = "ErrorCount"
    namespace = "CustomMetrics/${var.project_name}"
    value     = "1"
  }
}

# Only create detailed metrics in production
resource "aws_cloudwatch_log_metric_filter" "api_latency" {
  count          = var.environment == "prod" ? 1 : 0
  name           = "${var.project_name}-${var.environment}-api-latency-filter"
  pattern        = "[responseLatency, latency]"
  log_group_name = aws_cloudwatch_log_group.api_gateway_logs.name

  metric_transformation {
    name      = "APILatency"
    namespace = "CustomMetrics/${var.project_name}"
    value     = "$latency"
  }
}
```

### Dashboard Optimization

```hcl
# Create detailed dashboards only in production
resource "aws_cloudwatch_dashboard" "detailed" {
  count          = var.environment == "prod" ? 1 : 0
  dashboard_name = "${var.project_name}-${var.environment}-detailed-dashboard"

  # ... detailed dashboard configuration ...
}

# Create a simpler dashboard for non-production environments
resource "aws_cloudwatch_dashboard" "simple" {
  count          = var.environment == "prod" ? 0 : 1
  dashboard_name = "${var.project_name}-${var.environment}-dashboard"

  # ... simpler dashboard configuration ...
}
```

## Operational Readiness

### Health Check Lambda

```hcl
# Lambda function for health checks
resource "aws_lambda_function" "health_check" {
  function_name    = "${var.project_name}-${var.environment}-health-check"
  filename         = "health_check_lambda.zip"
  source_code_hash = filebase64sha256("health_check_lambda.zip")
  handler          = "index.handler"
  runtime          = "nodejs18.x"
  role             = aws_iam_role.lambda_health_check_role.arn
  timeout          = 60

  environment {
    variables = {
      API_ENDPOINT = "https://${aws_api_gateway_rest_api.example.id}.execute-api.${var.aws_region}.amazonaws.com/${aws_api_gateway_stage.example.stage_name}"
      ENVIRONMENT  = var.environment
    }
  }

  tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}

# CloudWatch Event Rule to trigger health check every 5 minutes
resource "aws_cloudwatch_event_rule" "health_check" {
  name                = "${var.project_name}-${var.environment}-health-check"
  description         = "Trigger health check Lambda function"
  schedule_expression = "rate(5 minutes)"
}

resource "aws_cloudwatch_event_target" "health_check" {
  rule      = aws_cloudwatch_event_rule.health_check.name
  target_id = "TriggerHealthCheckLambda"
  arn       = aws_lambda_function.health_check.arn
}

resource "aws_lambda_permission" "allow_cloudwatch_to_call_health_check" {
  statement_id  = "AllowExecutionFromCloudWatch"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.health_check.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.health_check.arn
}

# Alarm for failed health checks
resource "aws_cloudwatch_metric_alarm" "health_check_failures" {
  alarm_name          = "${var.project_name}-${var.environment}-health-check-failures"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  period              = 300
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "This metric monitors health check Lambda failures"

  dimensions = {
    FunctionName = aws_lambda_function.health_check.function_name
  }

  alarm_actions = [aws_sns_topic.critical_alerts.arn]
  ok_actions    = [aws_sns_topic.info_alerts.arn]
}
```

### Synthetic Monitoring

```hcl
# CloudWatch Synthetics Canary
resource "aws_synthetics_canary" "api_canary" {
  name                 = "${var.project_name}-${var.environment}-api-canary"
  artifact_s3_location = "s3://${aws_s3_bucket.monitoring_artifacts.bucket}/canary-artifacts/"
  execution_role_arn   = aws_iam_role.synthetics_role.arn
  handler              = "index.handler"
  runtime_version      = "syn-nodejs-puppeteer-3.9"
  start_canary         = true

  schedule {
    expression = "rate(5 minutes)"
  }

  run_config {
    timeout_in_seconds = 60
    memory_in_mb       = 1024
    active_tracing     = true
  }

  code {
    handler  = "index.handler"
    zip_file = filebase64("canary_script.zip")
  }

  tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}

# Alarm for Canary failures
resource "aws_cloudwatch_metric_alarm" "canary_failures" {
  alarm_name          = "${var.project_name}-${var.environment}-canary-failures"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "Failed"
  namespace           = "CloudWatchSynthetics"
  period              = 300
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "This metric monitors Synthetics Canary failures"

  dimensions = {
    CanaryName = aws_synthetics_canary.api_canary.name
  }

  alarm_actions = [aws_sns_topic.critical_alerts.arn]
  ok_actions    = [aws_sns_topic.info_alerts.arn]
}
```

### Service Health Dashboard

```hcl
resource "aws_cloudwatch_dashboard" "service_health" {
  dashboard_name = "${var.project_name}-${var.environment}-service-health"

  dashboard_body = jsonencode({
    widgets = [
      # Service Health Status
      {
        type   = "text",
        x      = 0,
        y      = 0,
        width  = 24,
        height = 1,
        properties = {
          markdown = "# Service Health Dashboard - ${var.environment} Environment"
        }
      },
      # API Gateway Health
      {
        type   = "metric",
        x      = 0,
        y      = 1,
        width  = 12,
        height = 6,
        properties = {
          metrics = [
            ["AWS/ApiGateway", "5XXError", "ApiName", aws_api_gateway_rest_api.example.name, "Stage", aws_api_gateway_stage.example.stage_name, { stat = "Sum" }],
            ["AWS/ApiGateway", "4XXError", "ApiName", aws_api_gateway_rest_api.example.name, "Stage", aws_api_gateway_stage.example.stage_name, { stat = "Sum" }]
          ],
          view    = "timeSeries",
          stacked = false,
          region  = var.aws_region,
          title   = "API Gateway Errors",
          period  = 300
        }
      },
      # Lambda Health
      {
        type   =
