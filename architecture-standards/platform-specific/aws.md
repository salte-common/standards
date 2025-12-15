# AWS Architecture Standards

## Overview

This document outlines AWS-specific architectural standards that prioritize cost-effective, highly-available serverless resources. These standards ensure scalable, maintainable, and secure applications while minimizing operational overhead and infrastructure costs.

## Core AWS Principles

### 1. Serverless-First Approach
- **Prefer Serverless**: Choose serverless services over managed services when possible
- **Pay-per-Use**: Leverage services that charge only for actual usage
- **Auto-scaling**: Utilize services that automatically scale based on demand
- **Managed Operations**: Minimize operational overhead through managed services

### 2. Network Security and Isolation
- **Private Subnet Deployment**: All compute resources must be deployed to private subnets
- **Internet Exposure Control**: Only API Gateway, CloudFront, and Application Load Balancers should be internet-facing
- **VPC Endpoints**: Use VPC endpoints for AWS service access to avoid internet traffic
- **Security Groups**: Implement restrictive security groups for all resources
- **NAT Gateways**: Use NAT gateways for outbound internet access from private subnets

### 3. Cost Optimization
- **Right-sizing**: Use appropriate instance types and service tiers
- **Reserved Capacity**: Leverage reserved instances and savings plans for predictable workloads
- **Spot Instances**: Use spot instances for fault-tolerant, flexible workloads
- **Resource Tagging**: Implement comprehensive tagging for cost allocation

### 4. High Availability and Fault Tolerance
- **Multi-AZ Deployment**: Deploy across multiple Availability Zones
- **Regional Redundancy**: Consider cross-region deployment for critical applications
- **Graceful Degradation**: Design systems to handle partial failures
- **Health Checks**: Implement comprehensive health monitoring

## Serverless Architecture Patterns

### Single Page Application (SPA) Architecture

#### Recommended Pattern
```
Route 53 → CloudFront → S3 (Static Hosting)
                    ↓
                API Gateway → Lambda → DynamoDB/RDS
                    ↓
                CloudWatch (Logging & Monitoring)
```

#### Implementation Standards
- **DNS Management**: Use Route 53 for domain management and health checks
- **Content Delivery**: CloudFront for global content distribution and caching (internet-facing)
- **Static Hosting**: S3 for static assets with website hosting enabled
- **API Layer**: API Gateway for RESTful API management and throttling (internet-facing)
- **Compute**: Lambda for serverless compute deployed to private subnets with auto-scaling
- **Database**: DynamoDB for NoSQL or RDS for relational data in private subnets
- **Monitoring**: CloudWatch for logs, metrics, and alerting
- **Network Security**: VPC endpoints for AWS service access, NAT gateways for outbound internet

#### SPA Configuration Example
```hcl
# Terraform configuration for SPA architecture
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# S3 Bucket for static hosting
resource "aws_s3_bucket" "website" {
  bucket = "${var.application_name}-website-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_website_configuration" "website" {
  bucket = aws_s3_bucket.website.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "error.html"
  }
}

resource "aws_s3_bucket_public_access_block" "website" {
  bucket = aws_s3_bucket.website.id

  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

resource "aws_s3_bucket_policy" "website" {
  bucket = aws_s3_bucket.website.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "PublicReadGetObject"
        Effect    = "Allow"
        Principal = "*"
        Action    = "s3:GetObject"
        Resource  = "${aws_s3_bucket.website.arn}/*"
      },
    ]
  })
}

# CloudFront Origin Access Identity
resource "aws_cloudfront_origin_access_identity" "website" {
  comment = "OAI for ${var.application_name} website"
}

# CloudFront Distribution
resource "aws_cloudfront_distribution" "website" {
  origin {
    domain_name = aws_s3_bucket.website.bucket_regional_domain_name
    origin_id   = "S3Origin"

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.website.cloudfront_access_identity_path
    }
  }

  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"

  default_cache_behavior {
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3Origin"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 3600
    max_ttl                = 86400
    compress               = true
  }

  price_class = "PriceClass_100"

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}

# API Gateway
resource "aws_api_gateway_rest_api" "api" {
  name        = "${var.application_name}-api"
  description = "Serverless API for SPA backend"
}

# Lambda Function
resource "aws_lambda_function" "api_handler" {
  filename         = "lambda_function.zip"
  function_name    = "${var.application_name}-api-handler"
  role            = aws_iam_role.lambda_execution_role.arn
  handler         = "index.handler"
  runtime         = "nodejs18.x"
  timeout         = 30
  memory_size     = 128

  environment {
    variables = {
      LOG_LEVEL = "INFO"
    }
  }

  source_code_hash = filebase64sha256("lambda_function.zip")
}

# IAM Role for Lambda
resource "aws_iam_role" "lambda_execution_role" {
  name = "${var.application_name}-lambda-execution-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# Variables
variable "application_name" {
  description = "Name of the application"
  type        = string
}

# Data sources
data "aws_caller_identity" "current" {}
```

### Microservices Architecture

#### Service Pattern
```
API Gateway → Lambda → DynamoDB/RDS
     ↓
CloudWatch (Logs & Metrics)
     ↓
X-Ray (Distributed Tracing)
```

#### Service Standards
- **API Gateway**: Use for service discovery, authentication, and rate limiting (internet-facing)
- **Lambda**: Implement business logic deployed to private subnets with appropriate timeout and memory settings
- **Database**: Choose based on data requirements (DynamoDB for NoSQL, RDS for relational) in private subnets
- **Monitoring**: Comprehensive logging and metrics collection
- **Tracing**: Use X-Ray for distributed tracing across services
- **Network Security**: VPC endpoints for AWS service access, security groups for resource isolation

#### Microservice Configuration
```hcl
# Lambda function deployed to private subnets
resource "aws_lambda_function" "microservice" {
  filename         = "${var.service_name}-${var.version}.zip"
  function_name    = "${var.service_name}-${var.environment}"
  role            = aws_iam_role.lambda_execution_role.arn
  handler         = "index.handler"
  runtime         = "nodejs18.x"
  timeout         = 30
  memory_size     = 256
  reserved_concurrent_executions = 100

  vpc_config {
    subnet_ids         = data.aws_subnets.private.ids
    security_group_ids = [aws_security_group.lambda.id]
  }

  environment {
    variables = {
      SERVICE_NAME = var.service_name
      ENVIRONMENT  = var.environment
      LOG_LEVEL    = "INFO"
      POWERTOOLS_SERVICE_NAME = var.service_name
    }
  }

  source_code_hash = filebase64sha256("${var.service_name}-${var.version}.zip")

  tracing_config {
    mode = "Active"
  }

  layers = [
    "arn:aws:lambda:${data.aws_region.current.name}:017000801446:layer:AWSLambdaPowertoolsTypeScript:1"
  ]
}

# IAM Role for Lambda execution with VPC access
resource "aws_iam_role" "lambda_execution_role" {
  name = "${var.service_name}-lambda-execution-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy_attachment" "lambda_vpc" {
  role       = aws_iam_role.lambda_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}

resource "aws_iam_role_policy_attachment" "lambda_xray" {
  role       = aws_iam_role.lambda_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess"
}

# Variables
variable "service_name" {
  description = "Name of the microservice"
  type        = string
}

variable "environment" {
  description = "Environment (dev, qa, prod)"
  type        = string
}

variable "version" {
  description = "Version of the service"
  type        = string
}

variable "vpc_id" {
  description = "ID of the existing VPC"
  type        = string
}

# Data sources
data "aws_region" "current" {}

data "aws_vpc" "existing" {
  id = var.vpc_id
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.existing.id]
  }
  
  filter {
    name   = "tag:Type"
    values = ["private"]
  }
}

# Security group for Lambda functions
resource "aws_security_group" "lambda" {
  name_prefix = "${var.service_name}-lambda-${var.environment}"
  vpc_id      = data.aws_vpc.existing.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.service_name}-lambda-sg-${var.environment}"
  }
}
```

## Database Standards

### DynamoDB Standards
- **Single Table Design**: Prefer single table design for related data
- **Partition Key Design**: Design keys for even distribution and access patterns
- **GSI Usage**: Use Global Secondary Indexes for alternative access patterns
- **Auto-scaling**: Enable auto-scaling for read and write capacity
- **Point-in-Time Recovery**: Enable for critical tables
- **Encryption**: Use AWS managed keys for encryption at rest

### RDS Standards
- **Multi-AZ**: Deploy in Multi-AZ configuration for high availability
- **Read Replicas**: Use read replicas for read-heavy workloads
- **Backup Strategy**: Automated backups with point-in-time recovery
- **Maintenance Windows**: Schedule during low-traffic periods
- **Parameter Groups**: Use custom parameter groups for optimization

### Database Configuration Example
```hcl
# DynamoDB Table with best practices
resource "aws_dynamodb_table" "users" {
  name           = "${var.application_name}-users-${var.environment}"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "userId"

  attribute {
    name = "userId"
    type = "S"
  }

  attribute {
    name = "email"
    type = "S"
  }

  global_secondary_index {
    name     = "EmailIndex"
    hash_key = "email"

    projection_type = "ALL"
  }

  point_in_time_recovery {
    enabled = true
  }

  server_side_encryption {
    enabled = true
  }

  tags = {
    Environment = var.environment
    Application = var.application_name
  }
}

# Variables
variable "application_name" {
  description = "Name of the application"
  type        = string
}

variable "environment" {
  description = "Environment (dev, qa, prod)"
  type        = string
}
```

## Security Standards

### Identity and Access Management (IAM)
- **Principle of Least Privilege**: Grant minimal required permissions
- **Role-Based Access**: Use IAM roles for service-to-service communication
- **Cross-Account Access**: Use cross-account roles for multi-account setups
- **Temporary Credentials**: Use temporary credentials when possible
- **Access Reviews**: Regular access reviews and cleanup

### Network Security
- **VPC Configuration**: Deploy all compute resources to private subnets
- **Internet Exposure**: Only API Gateway, CloudFront, and Application Load Balancers should be internet-facing
- **Security Groups**: Restrict access with security groups
- **WAF Integration**: Use AWS WAF for API Gateway protection
- **DDoS Protection**: Enable Shield for critical applications
- **VPC Endpoints**: Use VPC endpoints for AWS service access
- **NAT Gateways**: Use NAT gateways for outbound internet access from private subnets

### Security Configuration Example
```hcl
# VPC Configuration for private compute resources
data "aws_vpc" "existing" {
  id = var.vpc_id
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.existing.id]
  }
  
  filter {
    name   = "tag:Type"
    values = ["private"]
  }
}

data "aws_subnets" "public" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.existing.id]
  }
  
  filter {
    name   = "tag:Type"
    values = ["public"]
  }
}

# Lambda function deployed to private subnets
resource "aws_lambda_function" "api_handler" {
  filename         = "lambda_function.zip"
  function_name    = "${var.application_name}-api-handler-${var.environment}"
  role            = aws_iam_role.lambda_execution_role.arn
  handler         = "index.handler"
  runtime         = "nodejs18.x"
  timeout         = 30
  memory_size     = 128

  vpc_config {
    subnet_ids         = data.aws_subnets.private.ids
    security_group_ids = [aws_security_group.lambda.id]
  }

  environment {
    variables = {
      LOG_LEVEL = "INFO"
    }
  }

  source_code_hash = filebase64sha256("lambda_function.zip")
}

# Security group for Lambda functions
resource "aws_security_group" "lambda" {
  name_prefix = "${var.application_name}-lambda-${var.environment}"
  vpc_id      = data.aws_vpc.existing.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.application_name}-lambda-sg-${var.environment}"
  }
}

# IAM Role with VPC permissions
resource "aws_iam_role" "lambda_execution_role" {
  name = "${var.application_name}-lambda-role-${var.environment}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy_attachment" "lambda_vpc" {
  role       = aws_iam_role.lambda_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}

resource "aws_iam_role_policy" "dynamodb_access" {
  name = "DynamoDBAccess"
  role = aws_iam_role.lambda_execution_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:UpdateItem",
          "dynamodb:DeleteItem",
          "dynamodb:Query",
          "dynamodb:Scan"
        ]
        Resource = aws_dynamodb_table.users.arn
      }
    ]
  })
}

# VPC Endpoints for AWS services (optional, for enhanced security)
resource "aws_vpc_endpoint" "dynamodb" {
  vpc_id       = data.aws_vpc.existing.id
  service_name = "com.amazonaws.${data.aws_region.current.name}.dynamodb"
  subnet_ids   = data.aws_subnets.private.ids

  tags = {
    Name = "${var.application_name}-dynamodb-endpoint-${var.environment}"
  }
}

resource "aws_vpc_endpoint" "logs" {
  vpc_id            = data.aws_vpc.existing.id
  service_name      = "com.amazonaws.${data.aws_region.current.name}.logs"
  subnet_ids        = data.aws_subnets.private.ids
  vpc_endpoint_type = "Interface"

  security_group_ids = [aws_security_group.vpc_endpoint.id]

  private_dns_enabled = true

  tags = {
    Name = "${var.application_name}-logs-endpoint-${var.environment}"
  }
}

# Security group for VPC endpoints
resource "aws_security_group" "vpc_endpoint" {
  name_prefix = "${var.application_name}-vpc-endpoint-${var.environment}"
  vpc_id      = data.aws_vpc.existing.id

  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = [aws_security_group.lambda.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.application_name}-vpc-endpoint-sg-${var.environment}"
  }
}

# Variables
variable "application_name" {
  description = "Name of the application"
  type        = string
}

variable "environment" {
  description = "Environment (dev, qa, prod)"
  type        = string
}

variable "vpc_id" {
  description = "ID of the existing VPC"
  type        = string
}

# Data sources
data "aws_region" "current" {}
```

## Monitoring and Observability

### CloudWatch Standards
- **Structured Logging**: Use JSON format for all logs
- **Log Groups**: Organize logs by service and environment
- **Metric Filters**: Create filters for business metrics
- **Dashboards**: Create operational dashboards
- **Alarms**: Set up alarms for critical metrics

### Alerting Standards
- **4XX/5XX Errors**: Alert when error rates exceed thresholds
- **Latency**: Monitor API response times
- **Availability**: Track service uptime
- **Cost**: Monitor spending and usage patterns
- **Security**: Alert on security events

### Monitoring Configuration Example
```hcl
# CloudWatch Alarms for API Gateway
resource "aws_cloudwatch_metric_alarm" "api_gateway_4xx" {
  alarm_name          = "${var.application_name}-4xx-errors-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "4XXError"
  namespace           = "AWS/ApiGateway"
  period              = "300"
  statistic           = "Sum"
  threshold           = "10"
  alarm_description   = "Alert when 4XX errors exceed threshold"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    ApiName = aws_api_gateway_rest_api.api.name
  }
}

resource "aws_cloudwatch_metric_alarm" "api_gateway_5xx" {
  alarm_name          = "${var.application_name}-5xx-errors-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "1"
  metric_name         = "5XXError"
  namespace           = "AWS/ApiGateway"
  period              = "300"
  statistic           = "Sum"
  threshold           = "5"
  alarm_description   = "Alert when 5XX errors exceed threshold"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    ApiName = aws_api_gateway_rest_api.api.name
  }
}

resource "aws_cloudwatch_metric_alarm" "lambda_errors" {
  alarm_name          = "${var.application_name}-lambda-errors-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  period              = "300"
  statistic           = "Sum"
  threshold           = "5"
  alarm_description   = "Alert when Lambda errors exceed threshold"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    FunctionName = aws_lambda_function.api_handler.function_name
  }
}

# SNS Topic for alerts
resource "aws_sns_topic" "alerts" {
  name = "${var.application_name}-alerts-${var.environment}"
}

# Variables
variable "application_name" {
  description = "Name of the application"
  type        = string
}

variable "environment" {
  description = "Environment (dev, qa, prod)"
  type        = string
}
```

## Cost Optimization Standards

### Resource Optimization
- **Lambda Configuration**: Right-size memory and timeout settings
- **DynamoDB Capacity**: Use on-demand billing for variable workloads
- **CloudFront Caching**: Optimize cache settings for content delivery
- **S3 Lifecycle**: Implement lifecycle policies for cost management
- **Reserved Capacity**: Use reserved instances for predictable workloads

### Cost Monitoring
- **Cost Allocation Tags**: Implement comprehensive tagging strategy
- **Cost Alerts**: Set up billing alerts for budget management
- **Usage Analysis**: Regular review of service usage patterns
- **Optimization Recommendations**: Act on AWS cost optimization recommendations

### Cost Optimization Example
```hcl
# S3 Lifecycle Policy for cost optimization
resource "aws_s3_bucket" "data" {
  bucket = "${var.application_name}-data-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_versioning" "data" {
  bucket = aws_s3_bucket.data.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "data" {
  bucket = aws_s3_bucket.data.id

  rule {
    id     = "TransitionToIA"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }
  }

  rule {
    id     = "DeleteOldVersions"
    status = "Enabled"

    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "GLACIER"
    }

    noncurrent_version_expiration {
      noncurrent_days = 365
    }
  }
}

# Variables
variable "application_name" {
  description = "Name of the application"
  type        = string
}

# Data sources
data "aws_caller_identity" "current" {}
```

## Deployment Standards

### Infrastructure as Code
- **Terraform**: Use Terraform for infrastructure definition and management
- **SAM**: Use Serverless Application Model for serverless applications
- **CDK**: Consider AWS CDK for complex infrastructure
- **Version Control**: Store infrastructure code in version control
- **Environment Parity**: Maintain consistency across environments

### CI/CD Pipeline
- **CodePipeline**: Use AWS CodePipeline for deployment automation
- **CodeBuild**: Use CodeBuild for build and test automation
- **CodeDeploy**: Use CodeDeploy for deployment orchestration
- **Blue-Green Deployment**: Implement blue-green deployment for zero-downtime updates
- **Rollback Strategy**: Automated rollback capabilities

### Deployment Configuration Example
```hcl
# CodePipeline for automated deployment
resource "aws_codepipeline" "main" {
  name     = "${var.application_name}-pipeline-${var.environment}"
  role_arn = aws_iam_role.codepipeline_role.arn

  artifact_store {
    location = aws_s3_bucket.artifact_store.bucket
    type     = "S3"
  }

  stage {
    name = "Source"

    action {
      name             = "Source"
      category         = "Source"
      owner            = "AWS"
      provider         = "CodeCommit"
      version          = "1"
      output_artifacts = ["source_output"]

      configuration = {
        RepositoryName = var.repository_name
        BranchName     = "main"
      }
    }
  }

  stage {
    name = "Build"

    action {
      name            = "Build"
      category        = "Build"
      owner           = "AWS"
      provider        = "CodeBuild"
      input_artifacts = ["source_output"]
      version         = "1"

      configuration = {
        ProjectName = aws_codebuild_project.main.name
      }
    }
  }

  stage {
    name = "Deploy"

    action {
      name            = "Deploy"
      category        = "Deploy"
      owner           = "AWS"
      provider        = "CloudFormation"
      input_artifacts = ["source_output"]
      version         = "1"

      configuration = {
        ActionMode     = "CREATE_UPDATE"
        StackName      = "${var.application_name}-${var.environment}"
        TemplatePath   = "source_output::template.yaml"
        RoleArn        = aws_iam_role.cloudformation_role.arn
        Capabilities   = "CAPABILITY_IAM"
      }
    }
  }
}

# S3 Bucket for artifacts
resource "aws_s3_bucket" "artifact_store" {
  bucket = "${var.application_name}-artifacts-${data.aws_caller_identity.current.account_id}"
}

# CodeBuild Project
resource "aws_codebuild_project" "main" {
  name          = "${var.application_name}-build-${var.environment}"
  description   = "Build project for ${var.application_name}"
  build_timeout = "10"
  service_role  = aws_iam_role.codebuild_role.arn

  artifacts {
    type = "CODEPIPELINE"
  }

  environment {
    compute_type                = "BUILD_GENERAL1_SMALL"
    image                       = "aws/codebuild/amazonlinux2-x86_64-standard:4.0"
    type                        = "LINUX_CONTAINER"
    image_pull_credentials_type = "CODEBUILD"
  }

  source {
    type = "CODEPIPELINE"
  }
}

# IAM Roles
resource "aws_iam_role" "codepipeline_role" {
  name = "${var.application_name}-codepipeline-role-${var.environment}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "codepipeline.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role" "codebuild_role" {
  name = "${var.application_name}-codebuild-role-${var.environment}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "codebuild.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role" "cloudformation_role" {
  name = "${var.application_name}-cloudformation-role-${var.environment}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "cloudformation.amazonaws.com"
        }
      }
    ]
  })
}

# Variables
variable "application_name" {
  description = "Name of the application"
  type        = string
}

variable "environment" {
  description = "Environment (dev, qa, prod)"
  type        = string
}

variable "repository_name" {
  description = "Name of the CodeCommit repository"
  type        = string
}

# Data sources
data "aws_caller_identity" "current" {}
```

## Performance Standards

### API Performance
- **Response Time**: Target < 200ms for API responses
- **Throughput**: Design for expected peak load
- **Caching**: Implement appropriate caching strategies
- **Connection Pooling**: Use connection pooling for database connections
- **Async Processing**: Use async patterns for long-running operations

### Scalability Patterns
- **Horizontal Scaling**: Design for horizontal scaling
- **Load Distribution**: Use load balancers for traffic distribution
- **Database Scaling**: Implement read replicas and sharding strategies
- **CDN Usage**: Use CloudFront for global content delivery
- **Auto-scaling**: Leverage auto-scaling capabilities

### Performance Configuration Example
```hcl
# API Gateway with performance optimizations
resource "aws_api_gateway_rest_api" "api" {
  name        = "${var.application_name}-api-${var.environment}"
  description = "High-performance serverless API"

  endpoint_configuration {
    types = ["REGIONAL"]
  }

  binary_media_types = [
    "application/json",
    "application/octet-stream"
  ]
}

resource "aws_api_gateway_resource" "proxy" {
  rest_api_id = aws_api_gateway_rest_api.api.id
  parent_id   = aws_api_gateway_rest_api.api.root_resource_id
  path_part   = "{proxy+}"
}

resource "aws_api_gateway_method" "proxy" {
  rest_api_id   = aws_api_gateway_rest_api.api.id
  resource_id   = aws_api_gateway_resource.proxy.id
  http_method   = "ANY"
  authorization = "NONE"

  request_parameters = {
    "method.request.path.proxy" = true
  }
}

resource "aws_api_gateway_integration" "lambda" {
  rest_api_id = aws_api_gateway_rest_api.api.id
  resource_id = aws_api_gateway_resource.proxy.id
  http_method = aws_api_gateway_method.proxy.http_method

  integration_http_method = "POST"
  type                   = "AWS_PROXY"
  uri                    = aws_lambda_function.api_handler.invoke_arn

  request_parameters = {
    "integration.request.path.proxy" = "method.request.path.proxy"
  }
}

resource "aws_api_gateway_deployment" "api" {
  depends_on = [
    aws_api_gateway_integration.lambda
  ]

  rest_api_id = aws_api_gateway_rest_api.api.id
  stage_name  = var.environment
}

# Lambda permission for API Gateway
resource "aws_lambda_permission" "apigw" {
  statement_id  = "AllowExecutionFromAPIGateway"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.api_handler.function_name
  principal     = "apigateway.amazonaws.com"

  source_arn = "${aws_api_gateway_rest_api.api.execution_arn}/*/*"
}

# Variables
variable "application_name" {
  description = "Name of the application"
  type        = string
}

variable "environment" {
  description = "Environment (dev, qa, prod)"
  type        = string
}
```

## Disaster Recovery Standards

### Backup Strategy
- **Automated Backups**: Enable automated backups for all data stores
- **Cross-Region Backup**: Store backups in different regions
- **Point-in-Time Recovery**: Enable point-in-time recovery for critical data
- **Backup Testing**: Regular backup restoration testing
- **Retention Policy**: Define appropriate retention periods

### Recovery Objectives
- **RTO (Recovery Time Objective)**: Define maximum acceptable downtime
- **RPO (Recovery Point Objective)**: Define maximum acceptable data loss
- **Failover Strategy**: Implement automated failover procedures
- **Data Replication**: Use cross-region replication for critical data
- **Monitoring**: Monitor backup and recovery processes

### Disaster Recovery Configuration Example
```hcl
# Cross-region backup configuration
resource "aws_s3_bucket" "backup" {
  bucket = "${var.application_name}-backup-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_versioning" "backup" {
  bucket = aws_s3_bucket.backup.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "backup" {
  bucket = aws_s3_bucket.backup.id

  rule {
    id     = "BackupRetention"
    status = "Enabled"

    expiration {
      days = 2555  # 7 years
    }
  }
}

# Cross-region replication
resource "aws_s3_bucket_replication_configuration" "backup" {
  depends_on = [aws_s3_bucket_versioning.backup]
  bucket     = aws_s3_bucket.backup.id
  role       = aws_iam_role.s3_replication_role.arn

  rule {
    id     = "CrossRegionReplication"
    status = "Enabled"

    destination {
      bucket = "arn:aws:s3:::${var.application_name}-backup-${var.backup_region}"
    }
  }
}

# IAM Role for S3 replication
resource "aws_iam_role" "s3_replication_role" {
  name = "${var.application_name}-s3-replication-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "s3.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "s3_replication_policy" {
  name = "S3ReplicationPolicy"
  role = aws_iam_role.s3_replication_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetReplicationConfiguration",
          "s3:ListBucket"
        ]
        Resource = aws_s3_bucket.backup.arn
      },
      {
        Effect = "Allow"
        Action = [
          "s3:GetObjectVersion",
          "s3:GetObjectVersionAcl"
        ]
        Resource = "${aws_s3_bucket.backup.arn}/*"
      },
      {
        Effect = "Allow"
        Action = [
          "s3:ReplicateObject",
          "s3:ReplicateDelete"
        ]
        Resource = "arn:aws:s3:::${var.application_name}-backup-${var.backup_region}/*"
      }
    ]
  })
}

# Variables
variable "application_name" {
  description = "Name of the application"
  type        = string
}

variable "backup_region" {
  description = "Region for backup replication"
  type        = string
}

# Data sources
data "aws_caller_identity" "current" {}
```

## Compliance and Governance

### Data Protection
- **Encryption**: Encrypt data at rest and in transit
- **Access Controls**: Implement appropriate access controls
- **Audit Logging**: Enable comprehensive audit logging
- **Data Classification**: Classify data based on sensitivity
- **Privacy Compliance**: Ensure compliance with privacy regulations

### Security Compliance
- **SOC Compliance**: Maintain SOC compliance for security controls
- **PCI DSS**: Implement PCI DSS controls for payment processing
- **HIPAA**: Ensure HIPAA compliance for healthcare data
- **GDPR**: Implement GDPR controls for EU data
- **Regular Audits**: Conduct regular security audits

## Implementation Checklist

### New AWS Application Checklist
- [ ] Serverless architecture designed and documented
- [ ] VPC configuration with private subnets for compute resources
- [ ] Network security controls implemented (security groups, VPC endpoints)
- [ ] Internet exposure limited to API Gateway, CloudFront, and ALB only
- [ ] Cost optimization strategies implemented
- [ ] Security controls configured (IAM, VPC, WAF)
- [ ] Monitoring and alerting configured
- [ ] Backup and disaster recovery plan implemented
- [ ] Infrastructure as code implemented
- [ ] CI/CD pipeline configured
- [ ] Performance requirements defined and tested
- [ ] Compliance requirements identified and implemented
- [ ] Documentation completed

### AWS Service Selection Checklist
- [ ] Serverless option available and suitable
- [ ] Can be deployed to private subnets (if compute resource)
- [ ] Network security requirements satisfied
- [ ] Cost analysis completed
- [ ] Performance requirements met
- [ ] Security requirements satisfied
- [ ] Integration with existing services
- [ ] Operational overhead considered
- [ ] Scaling capabilities evaluated
- [ ] Compliance requirements met

### Cost Optimization Checklist
- [ ] Right-sized resources configured
- [ ] Reserved capacity purchased for predictable workloads
- [ ] Spot instances used where appropriate
- [ ] Lifecycle policies implemented
- [ ] Cost allocation tags applied
- [ ] Cost monitoring and alerting configured
- [ ] Regular cost reviews scheduled
- [ ] Optimization recommendations implemented

---

## Conclusion

These AWS-specific architectural standards ensure cost-effective, highly-available, and secure applications while leveraging the full potential of AWS serverless services. Regular review and updates ensure these standards remain relevant as AWS services evolve and new best practices emerge.

## References

- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [AWS Serverless Documentation](https://docs.aws.amazon.com/serverless/)
- [AWS Cost Optimization](https://aws.amazon.com/cost-optimization/)
- [AWS Security Best Practices](https://aws.amazon.com/security/security-learning/)
- [AWS Performance Best Practices](https://aws.amazon.com/performance/)
