# Java with Spring Boot Lambda Guidelines for AWS

This document outlines the best practices for developing and deploying AWS Lambda functions using Java with Spring Boot and Terraform. These guidelines are specifically tailored for EpsiHack's Java-based serverless applications.

## Table of Contents

- [Development Environment Setup](#development-environment-setup)
- [Project Structure](#project-structure)
- [Spring Boot Configuration](#spring-boot-configuration)
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

- Java 11 or 17 (LTS versions recommended)
- Maven or Gradle
- AWS CLI
- AWS SAM CLI (for local testing)
- IDE with Spring Boot support (IntelliJ IDEA, Eclipse, VS Code)

### AWS SDK Configuration

```xml
<!-- Maven dependencies -->
<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-lambda-java-core</artifactId>
    <version>1.2.2</version>
</dependency>
<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-lambda-java-events</artifactId>
    <version>3.11.1</version>
</dependency>
<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-lambda-java-log4j2</artifactId>
    <version>1.5.1</version>
</dependency>
```

## Project Structure

### Recommended Structure

```
project/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── epsihack/
│   │   │           ├── Application.java
│   │   │           ├── config/
│   │   │           ├── handler/
│   │   │           │   └── LambdaHandler.java
│   │   │           ├── service/
│   │   │           └── model/
│   │   └── resources/
│   │       ├── application.yml
│   │       └── application-prod.yml
│   └── test/
│       └── java/
│           └── com/
│               └── epsihack/
│                   └── handler/
│                       └── LambdaHandlerTest.java
├── pom.xml
└── terraform/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

## Spring Boot Configuration

### Minimal Spring Boot Application

```java
package com.epsihack;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@SpringBootApplication
public class Application {
    
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### Application Properties

```yaml
# application.yml
spring:
  application:
    name: lambda-function
  profiles:
    active: ${ENVIRONMENT:dev}

# Disable web server since we're running in Lambda
spring.main.web-application-type: none

# Configure logging
logging:
  level:
    root: INFO
    com.epsihack: DEBUG
    org.springframework: WARN
```

## Handler Implementation

### Basic Lambda Handler

```java
package com.epsihack.handler;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyRequestEvent;
import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyResponseEvent;
import org.springframework.stereotype.Component;

@Component
public class LambdaHandler implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {

    @Override
    public APIGatewayProxyResponseEvent handleRequest(APIGatewayProxyRequestEvent input, Context context) {
        context.getLogger().log("Received event: " + input);
        
        APIGatewayProxyResponseEvent response = new APIGatewayProxyResponseEvent();
        response.setStatusCode(200);
        response.setBody("Hello from Spring Boot Lambda!");
        
        return response;
    }
}
```

### Spring Cloud Function Approach

```java
package com.epsihack.handler;

import org.springframework.cloud.function.adapter.aws.SpringBootRequestHandler;
import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyRequestEvent;
import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyResponseEvent;

public class SpringCloudFunctionHandler extends SpringBootRequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {
    // Spring Cloud Function will automatically wire up your functions
}

// Define your function as a bean
@Bean
public Function<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> handleRequest() {
    return (input) -> {
        APIGatewayProxyResponseEvent response = new APIGatewayProxyResponseEvent();
        response.setStatusCode(200);
        response.setBody("Hello from Spring Cloud Function!");
        return response;
    };
}
```

## Dependency Management

### Maven Dependencies

```xml
<dependencies>
    <!-- Spring Boot -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    
    <!-- AWS Lambda Core -->
    <dependency>
        <groupId>com.amazonaws</groupId>
        <artifactId>aws-lambda-java-core</artifactId>
        <version>1.2.2</version>
    </dependency>
    
    <!-- AWS Lambda Events -->
    <dependency>
        <groupId>com.amazonaws</groupId>
        <artifactId>aws-lambda-java-events</artifactId>
        <version>3.11.1</version>
    </dependency>
    
    <!-- Spring Cloud Function (Optional) -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-function-adapter-aws</artifactId>
    </dependency>
    
    <!-- JSON Processing -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
    
    <!-- Testing -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Dependency Management for Lambda

```xml
<build>
    <plugins>
        <!-- Spring Boot Maven Plugin -->
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <classifier>aws</classifier>
                <mainClass>com.epsihack.Application</mainClass>
            </configuration>
        </plugin>
        
        <!-- Maven Shade Plugin for creating an uber-jar -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <version>3.3.0</version>
            <configuration>
                <createDependencyReducedPom>false</createDependencyReducedPom>
            </configuration>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>shade</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

## Building and Packaging

### Creating a Lambda-Compatible JAR

```bash
# Using Maven
mvn clean package

# Using Gradle
./gradlew clean build
```

### Optimizing JAR Size

1. **Use Spring Boot Thin Launcher**:
   ```xml
   <dependency>
       <groupId>org.springframework.boot.experimental</groupId>
       <artifactId>spring-boot-thin-launcher</artifactId>
       <version>1.0.28.RELEASE</version>
   </dependency>
   ```

2. **Exclude unnecessary dependencies**:
   ```xml
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter</artifactId>
       <exclusions>
           <exclusion>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-logging</artifactId>
           </exclusion>
       </exclusions>
   </dependency>
   ```

3. **Use ProGuard or R8 for minimization**:
   ```xml
   <plugin>
       <groupId>com.github.wvengen</groupId>
       <artifactId>proguard-maven-plugin</artifactId>
       <version>2.6.0</version>
       <executions>
           <execution>
               <phase>package</phase>
               <goals>
                   <goal>proguard</goal>
               </goals>
           </execution>
       </executions>
       <configuration>
           <!-- ProGuard configuration -->
       </configuration>
   </plugin>
   ```

## Testing

### Unit Testing

```java
package com.epsihack.handler;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyRequestEvent;
import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyResponseEvent;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.mockito.Mockito.when;

@SpringBootTest
public class LambdaHandlerTest {

    @Autowired
    private LambdaHandler handler;
    
    @MockBean
    private Context context;
    
    @Test
    public void testHandleRequest() {
        APIGatewayProxyRequestEvent request = new APIGatewayProxyRequestEvent();
        request.setBody("{}");
        
        APIGatewayProxyResponseEvent response = handler.handleRequest(request, context);
        
        assertEquals(200, response.getStatusCode());
        assertEquals("Hello from Spring Boot Lambda!", response.getBody());
    }
}
```

### Local Testing with AWS SAM

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Resources:
  JavaFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: com.epsihack.handler.LambdaHandler::handleRequest
      Runtime: java11
      CodeUri: target/lambda-function-1.0.0-aws.jar
      MemorySize: 512
      Timeout: 30
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /hello
            Method: get
```

```bash
# Build and run locally
mvn clean package
sam local start-api
```

## Deployment with Terraform

### Lambda Function Resource

```hcl
resource "aws_lambda_function" "java_function" {
  function_name    = "${var.project_name}-${var.environment}-java-function"
  filename         = "../target/lambda-function-1.0.0-aws.jar"
  source_code_hash = filebase64sha256("../target/lambda-function-1.0.0-aws.jar")
  handler          = "com.epsihack.handler.LambdaHandler::handleRequest"
  runtime          = "java11"
  memory_size      = 512
  timeout          = 30
  
  role = aws_iam_role.lambda_execution_role.arn
  
  environment {
    variables = {
      ENVIRONMENT = var.environment
      LOG_LEVEL   = var.environment == "prod" ? "INFO" : "DEBUG"
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
    Runtime     = "java11"
    Framework   = "spring-boot"
  }
}
```

### Layer for Common Dependencies

```hcl
resource "aws_lambda_layer_version" "java_dependencies" {
  layer_name = "${var.project_name}-${var.environment}-java-dependencies"
  filename   = "../layers/java-dependencies.zip"
  
  compatible_runtimes = ["java11"]
  
  source_code_hash = filebase64sha256("../layers/java-dependencies.zip")
}

# Reference the layer in your Lambda function
resource "aws_lambda_function" "java_function" {
  # ... other configuration ...
  layers = [aws_lambda_layer_version.java_dependencies.arn]
}
```

## Performance Optimization

### Cold Start Mitigation

1. **Use Provisioned Concurrency**:
   ```hcl
   resource "aws_lambda_provisioned_concurrency_config" "java_function" {
     function_name                     = aws_lambda_function.java_function.function_name
     provisioned_concurrent_executions = 5
     qualifier                         = aws_lambda_alias.java_function_alias.name
   }
   ```

2. **Optimize Spring Boot Configuration**:
   ```java
   @SpringBootApplication(proxyBeanMethods = false)
   @Import({MinimalConfiguration.class})
   public class Application {
       // Minimal configuration
   }
   ```

3. **Use GraalVM Native Image** (for advanced users):
   ```xml
   <plugin>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-maven-plugin</artifactId>
       <configuration>
           <image>
               <builder>paketobuildpacks/builder:tiny</builder>
               <env>
                   <BP_NATIVE_IMAGE>true</BP_NATIVE_IMAGE>
               </env>
           </image>
       </configuration>
   </plugin>
   ```

### Memory Allocation

Allocate sufficient memory for Java applications:

```hcl
resource "aws_lambda_function" "java_function" {
  # ... other configuration ...
  memory_size = 1024  # Minimum recommended for Java Spring Boot
}
```

## Logging and Monitoring

### Structured Logging

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class LambdaHandler implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {
    private static final Logger logger = LoggerFactory.getLogger(LambdaHandler.class);
    
    @Override
    public APIGatewayProxyResponseEvent handleRequest(APIGatewayProxyRequestEvent input, Context context) {
        logger.info("Processing request: {}", input);
        
        // Business logic
        
        if (error) {
            logger.error("Error processing request: {}", error.getMessage(), error);
        }
        
        logger.info("Request processed successfully");
        return response;
    }
}
```

### CloudWatch Metrics

```java
import com.amazonaws.services.cloudwatch.AmazonCloudWatch;
import com.amazonaws.services.cloudwatch.AmazonCloudWatchClientBuilder;
import com.amazonaws.services.cloudwatch.model.Dimension;
import com.amazonaws.services.cloudwatch.model.MetricDatum;
import com.amazonaws.services.cloudwatch.model.PutMetricDataRequest;
import com.amazonaws.services.cloudwatch.model.StandardUnit;

public class MetricsService {
    private final AmazonCloudWatch cloudWatch = AmazonCloudWatchClientBuilder.defaultClient();
    
    public void recordMetric(String metricName, double value, String unit) {
        Dimension dimension = new Dimension()
            .withName("Environment")
            .withValue(System.getenv("ENVIRONMENT"));
            
        MetricDatum datum = new MetricDatum()
            .withMetricName(metricName)
            .withUnit(unit)
            .withValue(value)
            .withDimensions(dimension);
            
        PutMetricDataRequest request = new PutMetricDataRequest()
            .withNamespace("EpsiHack/Lambda")
            .withMetricData(datum);
            
        cloudWatch.putMetricData(request);
    }
}
```

## Common Patterns and Examples

### API Gateway Integration

```java
public class ApiGatewayHandler implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Autowired
    private UserService userService;
    
    @Override
    public APIGatewayProxyResponseEvent handleRequest(APIGatewayProxyRequestEvent input, Context context) {
        APIGatewayProxyResponseEvent response = new APIGatewayProxyResponseEvent();
        response.setHeaders(Map.of("Content-Type", "application/json"));
        
        try {
            String path = input.getPath();
            String httpMethod = input.getHttpMethod();
            
            if ("/users".equals(path) && "GET".equals(httpMethod)) {
                List<User> users = userService.getAllUsers();
                response.setStatusCode(200);
                response.setBody(objectMapper.writeValueAsString(users));
            } else if (path.matches("/users/\\d+") && "GET".equals(httpMethod)) {
                String userId = path.substring(path.lastIndexOf('/') + 1);
                User user = userService.getUserById(userId);
                
                if (user != null) {
                    response.setStatusCode(200);
                    response.setBody(objectMapper.writeValueAsString(user));
                } else {
                    response.setStatusCode(404);
                    response.setBody("{\"error\":\"User not found\"}");
                }
            } else {
                response.setStatusCode(404);
                response.setBody("{\"error\":\"Not found\"}");
            }
        } catch (Exception e) {
            response.setStatusCode(500);
            response.setBody("{\"error\":\"Internal server error\"}");
        }
        
        return response;
    }
}
```

### Event Processing

```java
public class SQSEventHandler implements RequestHandler<SQSEvent, Void> {
    
    @Autowired
    private MessageProcessor messageProcessor;
    
    @Override
    public Void handleRequest(SQSEvent event, Context context) {
        for (SQSEvent.SQSMessage message : event.getRecords()) {
            try {
                messageProcessor.processMessage(message.getBody());
            } catch (Exception e) {
                context.getLogger().log("Error processing message: " + e.getMessage());
                // Consider implementing a dead-letter queue strategy
            }
        }
        return null;
    }
}
```

### DynamoDB Integration

```java
@Service
public class DynamoDBService {
    
    private final DynamoDbClient dynamoDbClient;
    private final String tableName;
    
    public DynamoDBService() {
        this.dynamoDbClient = DynamoDbClient.builder().build();
        this.tableName = System.getenv("DYNAMODB_TABLE");
    }
    
    public void saveItem(Map<String, AttributeValue> item) {
        PutItemRequest request = PutItemRequest.builder()
            .tableName(tableName)
            .item(item)
            .build();
            
        dynamoDbClient.putItem(request);
    }
    
    public Map<String, AttributeValue> getItem(String partitionKey, String partitionValue) {
        Map<String, AttributeValue> key = Map.of(
            partitionKey, AttributeValue.builder().s(partitionValue).build()
        );
        
        GetItemRequest request = GetItemRequest.builder()
            .tableName(tableName)
            .key(key)
            .build();
            
        GetItemResponse response = dynamoDbClient.getItem(request);
        return response.item();
    }
}
```

### Scheduled Tasks

```java
public class ScheduledTaskHandler implements RequestHandler<ScheduledEvent, Void> {
    
    @Autowired
    private TaskService taskService;
    
    @Override
    public Void handleRequest(ScheduledEvent event, Context context) {
        context.getLogger().log("Executing scheduled task at: " + event.getTime());
        taskService.performDailyTask();
        return null;
    }
}
```

This document provides guidelines for developing and deploying Java with Spring Boot Lambda functions using Terraform. For more information on general Lambda best practices, refer to the [Lambda Guidelines](lambda.md) document.
