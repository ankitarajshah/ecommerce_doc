# 🚀 DevOps & Deployment Guide

**Document Version:** 2.0  
**Last Updated:** March 26, 2026  
**Status:** ✅ Production Ready

---

## Table of Contents

1. [DevOps Overview](#devops-overview)
2. [CI/CD Pipeline](#cicd-pipeline)
3. [Infrastructure as Code](#infrastructure-as-code)
4. [Container Orchestration](#container-orchestration)
5. [Monitoring & Logging](#monitoring--logging)
6. [Deployment Strategies](#deployment-strategies)
7. [Disaster Recovery](#disaster-recovery)
8. [Performance Optimization](#performance-optimization)

---

## 1. DevOps Overview

### 1.1 DevOps Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   DevOps Pipeline                            │
├─────────────────────────────────────────────────────────────┤
│  Developer → Git Push → CI/CD → Build → Test →              │
│  Security Scan → Deploy → Monitor → Alert                   │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack

```yaml
Infrastructure:
  Cloud Provider: AWS
  Container: Docker
  Orchestration: AWS ECS Fargate
  Load Balancer: AWS ALB
  CDN: CloudFront
  DNS: Route 53

CI/CD:
  Version Control: GitHub
  CI/CD Platform: GitHub Actions
  Container Registry: AWS ECR
  Artifact Storage: AWS S3

Monitoring:
  Metrics: Prometheus + Grafana
  Logs: ELK Stack (Elasticsearch, Logstash, Kibana)
  APM: Datadog / New Relic
  Uptime: Pingdom / UptimeRobot
  Errors: Sentry

Infrastructure as Code:
  IaC Tool: Terraform
  Configuration: AWS CloudFormation
  Secrets: AWS Secrets Manager
  Parameters: AWS Parameter Store
```

---

## 2. CI/CD Pipeline

### 2.1 GitHub Actions Workflow

```yaml
# .github/workflows/deploy-production.yml
name: Deploy to Production

on:
  push:
    branches: [main]
  workflow_dispatch:

env:
  AWS_REGION: ap-south-1
  ECR_REPOSITORY: ecom-platform
  ECS_SERVICE: ecom-service
  ECS_CLUSTER: ecom-cluster
  ECS_TASK_DEFINITION: task-definition.json

jobs:
  # ============================================
  # Job 1: Run Tests
  # ============================================
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linting
        run: npm run lint
      
      - name: Run type checking
        run: npm run type-check
      
      - name: Setup database
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test
        run: |
          npx prisma migrate deploy
          npx prisma db seed
      
      - name: Run unit tests
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test
          REDIS_URL: redis://localhost:6379
        run: npm run test:unit
      
      - name: Run integration tests
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test
          REDIS_URL: redis://localhost:6379
        run: npm run test:integration
      
      - name: Generate coverage report
        run: npm run test:coverage
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json
          fail_ci_if_error: true

  # ============================================
  # Job 2: Security Scanning
  # ============================================
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: test
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Run Snyk security scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
      
      - name: Run npm audit
        run: npm audit --audit-level=moderate
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload Trivy results to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
      
      - name: Check for secrets
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD

  # ============================================
  # Job 3: Build and Push Docker Image
  # ============================================
  build:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    needs: [test, security]
    outputs:
      image: ${{ steps.build-image.outputs.image }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      
      - name: Build, tag, and push image to Amazon ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          # Build Docker image
          docker build \
            --build-arg NODE_ENV=production \
            --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
            --build-arg VCS_REF=${{ github.sha }} \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:latest \
            .
          
          # Push images
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
          
          # Output image URI
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT
      
      - name: Scan Docker image with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.build-image.outputs.image }}
          format: 'sarif'
          output: 'trivy-image-results.sarif'
      
      - name: Upload Trivy image scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-image-results.sarif'

  # ============================================
  # Job 4: Database Migration
  # ============================================
  migrate:
    name: Run Database Migrations
    runs-on: ubuntu-latest
    needs: build
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Get database URL from AWS Secrets Manager
        id: get-db-url
        run: |
          DB_URL=$(aws secretsmanager get-secret-value \
            --secret-id ecom-platform/prod/database \
            --query SecretString \
            --output text | jq -r .DATABASE_URL)
          echo "::add-mask::$DB_URL"
          echo "DATABASE_URL=$DB_URL" >> $GITHUB_ENV
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_REGION: ${{ env.AWS_REGION }}
      
      - name: Run Prisma migrations
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: ${{ env.DATABASE_URL }}

  # ============================================
  # Job 5: Deploy to ECS
  # ============================================
  deploy:
    name: Deploy to ECS
    runs-on: ubuntu-latest
    needs: [build, migrate]
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Download task definition
        run: |
          aws ecs describe-task-definition \
            --task-definition ${{ env.ECS_SERVICE }} \
            --query taskDefinition > task-definition.json
      
      - name: Fill in the new image ID in the task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: app
          image: ${{ needs.build.outputs.image }}
      
      - name: Deploy Amazon ECS task definition
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
      
      - name: Verify deployment
        run: |
          # Wait for service to stabilize
          sleep 30
          
          # Check health endpoint
          HEALTH_URL="https://api.ecomplatform.com/health"
          RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" $HEALTH_URL)
          
          if [ $RESPONSE -eq 200 ]; then
            echo "✅ Deployment successful!"
          else
            echo "❌ Deployment failed! Health check returned $RESPONSE"
            exit 1
          fi

  # ============================================
  # Job 6: Post-Deployment Tasks
  # ============================================
  post-deploy:
    name: Post-Deployment Tasks
    runs-on: ubuntu-latest
    needs: deploy
    
    steps:
      - name: Invalidate CloudFront cache
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/*"
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      
      - name: Notify Slack
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "✅ Production deployment successful!",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Production Deployment Successful* :rocket:\n\n*Commit:* ${{ github.sha }}\n*Author:* ${{ github.actor }}\n*Branch:* ${{ github.ref_name }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
      
      - name: Create Sentry release
        uses: getsentry/action-release@v1
        env:
          SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}
          SENTRY_ORG: ${{ secrets.SENTRY_ORG }}
          SENTRY_PROJECT: ecom-platform
        with:
          environment: production
          version: ${{ github.sha }}
      
      - name: Tag Docker image as stable
        run: |
          docker tag ${{ needs.build.outputs.image }} \
            ${{ secrets.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:stable
          docker push ${{ secrets.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:stable
```

### 2.2 Staging Deployment

```yaml
# .github/workflows/deploy-staging.yml
name: Deploy to Staging

on:
  push:
    branches: [develop]
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.ecomplatform.com
    
    steps:
      # Similar steps as production but with staging environment
      - name: Checkout code
        uses: actions/checkout@v3
      
      # ... (similar steps)
      
      - name: Deploy to staging ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ecom-service-staging
          cluster: ecom-cluster-staging
          wait-for-service-stability: true
      
      - name: Run smoke tests
        run: npm run test:smoke
        env:
          API_URL: https://staging-api.ecomplatform.com
      
      - name: Comment on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v6
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '✅ Deployed to staging: https://staging.ecomplatform.com'
            })
```

### 2.3 Rollback Workflow

```yaml
# .github/workflows/rollback.yml
name: Rollback Production

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version/Tag to rollback to'
        required: true
        type: string

jobs:
  rollback:
    name: Rollback to Previous Version
    runs-on: ubuntu-latest
    
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1
      
      - name: Get previous task definition
        run: |
          aws ecs describe-task-definition \
            --task-definition ecom-service:${{ inputs.version }} \
            --query taskDefinition > task-definition.json
      
      - name: Deploy previous version
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: task-definition.json
          service: ecom-service
          cluster: ecom-cluster
          wait-for-service-stability: true
      
      - name: Verify rollback
        run: |
          sleep 30
          HEALTH_URL="https://api.ecomplatform.com/health"
          RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" $HEALTH_URL)
          
          if [ $RESPONSE -eq 200 ]; then
            echo "✅ Rollback successful!"
          else
            echo "❌ Rollback verification failed!"
            exit 1
          fi
      
      - name: Notify team
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "⚠️ Production rollback to version ${{ inputs.version }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 3. Infrastructure as Code

### 3.1 Terraform Configuration

```hcl
# infrastructure/terraform/main.tf
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket         = "ecom-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "ap-south-1"
    encrypt        = true
    dynamodb_table = "terraform-lock"
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Environment = var.environment
      Project     = "ecom-platform"
      ManagedBy   = "Terraform"
    }
  }
}

# ============================================
# VPC Configuration
# ============================================
module "vpc" {
  source = "./modules/vpc"
  
  name               = "ecom-vpc-${var.environment}"
  cidr               = "10.0.0.0/16"
  availability_zones = ["ap-south-1a", "ap-south-1b", "ap-south-1c"]
  
  public_subnet_cidrs   = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  private_subnet_cidrs  = ["10.0.10.0/24", "10.0.11.0/24", "10.0.12.0/24"]
  database_subnet_cidrs = ["10.0.20.0/24", "10.0.21.0/24", "10.0.22.0/24"]
  
  enable_nat_gateway = true
  single_nat_gateway = false
  enable_dns_hostnames = true
  enable_dns_support   = true
}

# ============================================
# RDS PostgreSQL
# ============================================
module "database" {
  source = "./modules/rds"
  
  identifier     = "ecom-db-${var.environment}"
  engine         = "postgres"
  engine_version = "15.3"
  instance_class = "db.t4g.large"
  
  allocated_storage     = 100
  max_allocated_storage = 500
  storage_encrypted     = true
  
  db_name  = "ecommerce"
  username = "postgres"
  port     = 5432
  
  multi_az               = true
  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "Mon:04:00-Mon:05:00"
  
  vpc_security_group_ids = [aws_security_group.database.id]
  db_subnet_group_name   = module.vpc.database_subnet_group_name
  
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
  
  performance_insights_enabled = true
  monitoring_interval         = 60
  
  deletion_protection = true
  skip_final_snapshot = false
  final_snapshot_identifier = "ecom-db-final-snapshot-${formatdate("YYYY-MM-DD-hhmm", timestamp())}"
}

# ============================================
# ElastiCache Redis
# ============================================
module "redis" {
  source = "./modules/elasticache"
  
  cluster_id           = "ecom-redis-${var.environment}"
  engine               = "redis"
  engine_version       = "7.0"
  node_type            = "cache.t4g.medium"
  num_cache_nodes      = 2
  parameter_group_name = "default.redis7"
  port                 = 6379
  
  subnet_group_name  = module.vpc.elasticache_subnet_group_name
  security_group_ids = [aws_security_group.redis.id]
  
  automatic_failover_enabled = true
  multi_az_enabled          = true
  
  snapshot_retention_limit = 5
  snapshot_window         = "03:00-05:00"
  
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
}

# ============================================
# ECS Cluster
# ============================================
resource "aws_ecs_cluster" "main" {
  name = "ecom-cluster-${var.environment}"
  
  setting {
    name  = "containerInsights"
    value = "enabled"
  }
  
  configuration {
    execute_command_configuration {
      logging = "OVERRIDE"
      
      log_configuration {
        cloud_watch_log_group_name = aws_cloudwatch_log_group.ecs_exec.name
      }
    }
  }
}

resource "aws_ecs_cluster_capacity_providers" "main" {
  cluster_name = aws_ecs_cluster.main.name
  
  capacity_providers = ["FARGATE", "FARGATE_SPOT"]
  
  default_capacity_provider_strategy {
    base              = 1
    weight            = 100
    capacity_provider = "FARGATE"
  }
}

# ============================================
# ECS Task Definition
# ============================================
resource "aws_ecs_task_definition" "app" {
  family                   = "ecom-app-${var.environment}"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = "1024"
  memory                   = "2048"
  execution_role_arn       = aws_iam_role.ecs_execution_role.arn
  task_role_arn            = aws_iam_role.ecs_task_role.arn
  
  container_definitions = jsonencode([
    {
      name  = "app"
      image = "${aws_ecr_repository.app.repository_url}:latest"
      
      portMappings = [
        {
          containerPort = 3000
          protocol      = "tcp"
        }
      ]
      
      environment = [
        {
          name  = "NODE_ENV"
          value = var.environment
        },
        {
          name  = "PORT"
          value = "3000"
        }
      ]
      
      secrets = [
        {
          name      = "DATABASE_URL"
          valueFrom = "${aws_secretsmanager_secret.database.arn}:DATABASE_URL::"
        },
        {
          name      = "REDIS_URL"
          valueFrom = "${aws_secretsmanager_secret.redis.arn}:REDIS_URL::"
        },
        {
          name      = "JWT_SECRET"
          valueFrom = "${aws_secretsmanager_secret.app.arn}:JWT_SECRET::"
        }
      ]
      
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.app.name
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "ecs"
        }
      }
      
      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 60
      }
    }
  ])
}

# ============================================
# ECS Service
# ============================================
resource "aws_ecs_service" "app" {
  name            = "ecom-service-${var.environment}"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = var.desired_count
  launch_type     = "FARGATE"
  
  deployment_configuration {
    maximum_percent         = 200
    minimum_healthy_percent = 100
    
    deployment_circuit_breaker {
      enable   = true
      rollback = true
    }
  }
  
  network_configuration {
    subnets          = module.vpc.private_subnet_ids
    security_groups  = [aws_security_group.app.id]
    assign_public_ip = false
  }
  
  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "app"
    container_port   = 3000
  }
  
  service_registries {
    registry_arn = aws_service_discovery_service.app.arn
  }
  
  enable_execute_command = true
  
  depends_on = [aws_lb_listener.https]
}

# Auto Scaling
resource "aws_appautoscaling_target" "ecs" {
  max_capacity       = 10
  min_capacity       = 2
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.app.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "cpu" {
  name               = "cpu-autoscaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs.service_namespace
  
  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value = 70.0
  }
}

# ============================================
# Application Load Balancer
# ============================================
resource "aws_lb" "main" {
  name               = "ecom-alb-${var.environment}"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = module.vpc.public_subnet_ids
  
  enable_deletion_protection = true
  enable_http2              = true
  enable_cross_zone_load_balancing = true
  
  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = "alb"
    enabled = true
  }
}

resource "aws_lb_target_group" "app" {
  name     = "ecom-tg-${var.environment}"
  port     = 3000
  protocol = "HTTP"
  vpc_id   = module.vpc.vpc_id
  target_type = "ip"
  
  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
    path                = "/health"
    matcher             = "200"
  }
  
  deregistration_delay = 30
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.main.arn
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = "80"
  protocol          = "HTTP"
  
  default_action {
    type = "redirect"
    
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

# ============================================
# CloudFront Distribution
# ============================================
resource "aws_cloudfront_distribution" "main" {
  enabled             = true
  is_ipv6_enabled     = true
  comment             = "Ecom Platform ${var.environment}"
  default_root_object = "index.html"
  price_class         = "PriceClass_All"
  
  aliases = ["www.ecomplatform.com", "ecomplatform.com"]
  
  origin {
    domain_name = aws_lb.main.dns_name
    origin_id   = "alb"
    
    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }
  
  origin {
    domain_name = aws_s3_bucket.static.bucket_regional_domain_name
    origin_id   = "s3"
    
    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.main.cloudfront_access_identity_path
    }
  }
  
  default_cache_behavior {
    allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods         = ["GET", "HEAD", "OPTIONS"]
    target_origin_id       = "alb"
    viewer_protocol_policy = "redirect-to-https"
    compress               = true
    
    forwarded_values {
      query_string = true
      headers      = ["Host", "Authorization", "CloudFront-Forwarded-Proto"]
      
      cookies {
        forward = "all"
      }
    }
    
    min_ttl     = 0
    default_ttl = 0
    max_ttl     = 0
  }
  
  ordered_cache_behavior {
    path_pattern           = "/static/*"
    allowed_methods        = ["GET", "HEAD", "OPTIONS"]
    cached_methods         = ["GET", "HEAD", "OPTIONS"]
    target_origin_id       = "s3"
    viewer_protocol_policy = "redirect-to-https"
    compress               = true
    
    forwarded_values {
      query_string = false
      
      cookies {
        forward = "none"
      }
    }
    
    min_ttl     = 0
    default_ttl = 86400
    max_ttl     = 31536000
  }
  
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }
  
  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.main.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }
  
  web_acl_id = aws_wafv2_web_acl.main.arn
}

# ============================================
# WAF Configuration
# ============================================
resource "aws_wafv2_web_acl" "main" {
  name  = "ecom-waf-${var.environment}"
  scope = "CLOUDFRONT"
  
  default_action {
    allow {}
  }
  
  rule {
    name     = "RateLimitRule"
    priority = 1
    
    action {
      block {}
    }
    
    statement {
      rate_based_statement {
        limit              = 2000
        aggregate_key_type = "IP"
      }
    }
    
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimitRule"
      sampled_requests_enabled   = true
    }
  }
  
  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 2
    
    override_action {
      none {}
    }
    
    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }
    
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedRulesCommonRuleSetMetric"
      sampled_requests_enabled   = true
    }
  }
  
  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "ecom-waf-metric"
    sampled_requests_enabled   = true
  }
}

# ============================================
# Outputs
# ============================================
output "alb_dns_name" {
  value = aws_lb.main.dns_name
}

output "cloudfront_domain_name" {
  value = aws_cloudfront_distribution.main.domain_name
}

output "ecs_cluster_name" {
  value = aws_ecs_cluster.main.name
}

output "database_endpoint" {
  value     = module.database.endpoint
  sensitive = true
}

output "redis_endpoint" {
  value     = module.redis.endpoint
  sensitive = true
}
```

### 3.2 Variables Configuration

```hcl
# infrastructure/terraform/variables.tf
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-south-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "desired_count" {
  description = "Desired number of ECS tasks"
  type        = number
  default     = 2
}

variable "domain_name" {
  description = "Domain name"
  type        = string
  default     = "ecomplatform.com"
}

# Environment-specific tfvars files

# production.tfvars
environment    = "production"
desired_count  = 4
instance_class = "db.r6g.xlarge"

# staging.tfvars
environment    = "staging"
desired_count  = 2
instance_class = "db.t4g.large"
```

---

## 4. Container Orchestration

### 4.1 Dockerfile

```dockerfile
# Multi-stage build for optimized image size
# Stage 1: Dependencies
FROM node:20-alpine AS dependencies

WORKDIR /app

# Install dependencies for native modules
RUN apk add --no-cache python3 make g++

# Copy package files
COPY package*.json ./
COPY prisma ./prisma/

# Install dependencies
RUN npm ci --only=production && \
    npm cache clean --force

# Generate Prisma client
RUN npx prisma generate

# ============================================
# Stage 2: Build
# ============================================
FROM node:20-alpine AS build

WORKDIR /app

# Copy dependencies from previous stage
COPY --from=dependencies /app/node_modules ./node_modules
COPY --from=dependencies /app/prisma ./prisma

# Copy source code
COPY . .

# Build TypeScript
RUN npm run build

# ============================================
# Stage 3: Production
# ============================================
FROM node:20-alpine AS production

# Add non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Copy built application
COPY --from=build --chown=nodejs:nodejs /app/dist ./dist
COPY --from=build --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=build --chown=nodejs:nodejs /app/prisma ./prisma
COPY --from=build --chown=nodejs:nodejs /app/package*.json ./

# Install production tools
RUN apk add --no-cache curl

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Start application
CMD ["node", "dist/server.js"]
```

### 4.2 Docker Compose (Local Development)

```yaml
# docker-compose.yml
version: '3.8'

services:
  # ============================================
  # Application
  # ============================================
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: development
    container_name: ecom-app
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: development
      DATABASE_URL: postgresql://postgres:postgres@postgres:5432/ecommerce
      REDIS_URL: redis://redis:6379
    volumes:
      - ./src:/app/src
      - ./prisma:/app/prisma
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - ecom-network
    restart: unless-stopped

  # ============================================
  # PostgreSQL Database
  # ============================================
  postgres:
    image: postgres:15-alpine
    container_name: ecom-postgres
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: ecommerce
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - ecom-network
    restart: unless-stopped

  # ============================================
  # Redis Cache
  # ============================================
  redis:
    image: redis:7-alpine
    container_name: ecom-redis
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes --requirepass redispass
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    networks:
      - ecom-network
    restart: unless-stopped

  # ============================================
  # Elasticsearch
  # ============================================
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.9.0
    container_name: ecom-elasticsearch
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    networks:
      - ecom-network
    restart: unless-stopped

  # ============================================
  # Kibana
  # ============================================
  kibana:
    image: docker.elastic.co/kibana/kibana:8.9.0
    container_name: ecom-kibana
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200
    depends_on:
      - elasticsearch
    networks:
      - ecom-network
    restart: unless-stopped

  # ============================================
  # Prometheus
  # ============================================
  prometheus:
    image: prom/prometheus:latest
    container_name: ecom-prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    networks:
      - ecom-network
    restart: unless-stopped

  # ============================================
  # Grafana
  # ============================================
  grafana:
    image: grafana/grafana:latest
    container_name: ecom-grafana
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./monitoring/grafana/datasources:/etc/grafana/provisioning/datasources
    depends_on:
      - prometheus
    networks:
      - ecom-network
    restart: unless-stopped

  # ============================================
  # Mailhog (Email Testing)
  # ============================================
  mailhog:
    image: mailhog/mailhog:latest
    container_name: ecom-mailhog
    ports:
      - "1025:1025"  # SMTP
      - "8025:8025"  # Web UI
    networks:
      - ecom-network
    restart: unless-stopped

networks:
  ecom-network:
    driver: bridge

volumes:
  postgres-data:
  redis-data:
  elasticsearch-data:
  prometheus-data:
  grafana-data:
```

---

## 5. Monitoring & Logging

### 5.1 Prometheus Configuration

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'ecom-production'
    replica: '1'

# Alert manager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

# Load rules once and periodically evaluate them
rule_files:
  - "/etc/prometheus/rules/*.yml"

# Scrape configurations
scrape_configs:
  # Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Node.js application metrics
  - job_name: 'nodejs-app'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['app:3000']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        replacement: 'ecom-app'

  # PostgreSQL exporter
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']

  # Redis exporter
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']

  # AWS CloudWatch metrics
  - job_name: 'cloudwatch'
    static_configs:
      - targets: ['cloudwatch-exporter:9106']
```

### 5.2 Alert Rules

```yaml
# monitoring/rules/alerts.yml
groups:
  - name: application_alerts
    interval: 30s
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} req/s"

      # High response time
      - alert: HighResponseTime
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High response time"
          description: "P95 response time is {{ $value }}s"

      # Database connection pool exhausted
      - alert: DatabaseConnectionPoolExhausted
        expr: pg_stat_activity_count >= pg_settings_max_connections * 0.9
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Database connection pool almost exhausted"
          description: "{{ $value }} connections active"

      # High memory usage
      - alert: HighMemoryUsage
        expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage"
          description: "Memory usage is {{ $value | humanizePercentage }}"

      # High CPU usage
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage"
          description: "CPU usage is {{ $value }}%"

      # Service down
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} is down"
          description: "{{ $labels.instance }} has been down for more than 1 minute"

      # Redis unavailable
      - alert: RedisUnavailable
        expr: redis_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Redis is unavailable"
          description: "Redis has been unavailable for more than 1 minute"
```

### 5.3 Logging Configuration

```typescript
// src/config/logging.config.ts
import winston from 'winston';
import { ElasticsearchTransport } from 'winston-elasticsearch';

// Define log format
const logFormat = winston.format.combine(
  winston.format.timestamp(),
  winston.format.errors({ stack: true }),
  winston.format.metadata(),
  winston.format.json()
);

// Create logger instance
export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: logFormat,
  defaultMeta: {
    service: 'ecom-platform',
    environment: process.env.NODE_ENV
  },
  transports: [
    // Console transport
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      )
    }),

    // File transport - All logs
    new winston.transports.File({
      filename: 'logs/combined.log',
      maxsize: 10485760, // 10MB
      maxFiles: 10
    }),

    // File transport - Error logs
    new winston.transports.File({
      filename: 'logs/error.log',
      level: 'error',
      maxsize: 10485760,
      maxFiles: 10
    }),

    // Elasticsearch transport
    new ElasticsearchTransport({
      level: 'info',
      clientOpts: {
        node: process.env.ELASTICSEARCH_URL,
        auth: {
          username: process.env.ELASTICSEARCH_USERNAME!,
          password: process.env.ELASTICSEARCH_PASSWORD!
        }
      },
      index: 'ecom-logs',
      dataStream: true
    })
  ],

  // Handle exceptions and rejections
  exceptionHandlers: [
    new winston.transports.File({ filename: 'logs/exceptions.log' })
  ],
  rejectionHandlers: [
    new winston.transports.File({ filename: 'logs/rejections.log' })
  ]
});

// Request logging middleware
export function requestLogger(req: Request, res: Response, next: NextFunction) {
  const start = Date.now();

  res.on('finish', () => {
    const duration = Date.now() - start;
    
    logger.info('HTTP Request', {
      method: req.method,
      url: req.url,
      status: res.statusCode,
      duration,
      ip: req.ip,
      userAgent: req.get('user-agent'),
      userId: req.user?.userId
    });
  });

  next();
}
```

---

## 6. Deployment Strategies

### 6.1 Blue-Green Deployment

```typescript
// scripts/blue-green-deploy.ts
import { ECSClient, UpdateServiceCommand, DescribeServicesCommand } from '@aws-sdk/client-ecs';

async function blueGreenDeploy() {
  const ecs = new ECSClient({ region: 'ap-south-1' });
  
  const CLUSTER = 'ecom-cluster';
  const BLUE_SERVICE = 'ecom-service-blue';
  const GREEN_SERVICE = 'ecom-service-green';
  const TARGET_GROUP_ARN = process.env.TARGET_GROUP_ARN!;

  console.log('🚀 Starting Blue-Green Deployment');

  // Step 1: Deploy new version to green environment
  console.log('📦 Deploying to green environment...');
  await ecs.send(new UpdateServiceCommand({
    cluster: CLUSTER,
    service: GREEN_SERVICE,
    taskDefinition: 'ecom-app:latest',
    desiredCount: 2
  }));

  // Step 2: Wait for green to be stable
  console.log('⏳ Waiting for green environment to stabilize...');
  await waitForServiceStable(ecs, CLUSTER, GREEN_SERVICE);

  // Step 3: Run smoke tests
  console.log('🧪 Running smoke tests...');
  const testsPass = await runSmokeTests('https://green.ecomplatform.com');
  
  if (!testsPass) {
    console.error('❌ Smoke tests failed! Rolling back...');
    await rollbackDeployment(ecs, CLUSTER, GREEN_SERVICE);
    process.exit(1);
  }

  // Step 4: Switch traffic to green
  console.log('🔄 Switching traffic to green...');
  await switchTraffic(TARGET_GROUP_ARN, GREEN_SERVICE);

  // Step 5: Monitor for 10 minutes
  console.log('👀 Monitoring green environment...');
  await monitorEnvironment(10 * 60 * 1000);

  // Step 6: Scale down blue
  console.log('📉 Scaling down blue environment...');
  await ecs.send(new UpdateServiceCommand({
    cluster: CLUSTER,
    service: BLUE_SERVICE,
    desiredCount: 0
  }));

  console.log('✅ Blue-Green deployment completed successfully!');
}
```

### 6.2 Canary Deployment

```typescript
// scripts/canary-deploy.ts
async function canaryDeploy() {
  console.log('🐤 Starting Canary Deployment');

  // Step 1: Deploy canary with 10% traffic
  await deployCanary(10);
  await waitAndMonitor(5 * 60 * 1000); // 5 minutes

  // Step 2: Increase to 25% if metrics good
  if (await checkMetrics()) {
    await deployCanary(25);
    await waitAndMonitor(10 * 60 * 1000); // 10 minutes
  } else {
    await rollbackCanary();
    return;
  }

  // Step 3: Increase to 50%
  if (await checkMetrics()) {
    await deployCanary(50);
    await waitAndMonitor(15 * 60 * 1000); // 15 minutes
  } else {
    await rollbackCanary();
    return;
  }

  // Step 4: Full deployment
  if (await checkMetrics()) {
    await deployCanary(100);
    console.log('✅ Canary deployment completed successfully!');
  } else {
    await rollbackCanary();
  }
}
```

---

## 7. Disaster Recovery

### 7.1 Backup Strategy

```typescript
// src/services/backup/backup.service.ts
export class BackupService {
  // Automated database backups
  async backupDatabase(): Promise<void> {
    const timestamp = new Date().toISOString();
    const backupName = `ecom-db-backup-${timestamp}`;

    // Create RDS snapshot
    await rds.send(new CreateDBSnapshotCommand({
      DBSnapshotIdentifier: backupName,
      DBInstanceIdentifier: 'ecom-db-production'
    }));

    // Export to S3 for long-term storage
    await this.exportToS3(backupName);

    // Keep only last 30 days of backups
    await this.cleanupOldBackups(30);
  }

  // Backup Redis data
  async backupRedis(): Promise<void> {
    // Trigger Redis BGSAVE
    await redis.bgsave();

    // Copy RDB file to S3
    const rdbFile = '/var/lib/redis/dump.rdb';
    await s3.upload({
      Bucket: 'ecom-backups',
      Key: `redis/dump-${Date.now()}.rdb`,
      Body: fs.createReadStream(rdbFile)
    });
  }

  // Backup uploaded files
  async backupFiles(): Promise<void> {
    // Sync S3 buckets to backup bucket
    await s3Sync(
      'ecom-uploads',
      'ecom-uploads-backup',
      { deleteRemoved: false }
    );
  }
}

// Schedule backups
cron.schedule('0 2 * * *', async () => {
  await backupService.backupDatabase();
  await backupService.backupRedis();
  await backupService.backupFiles();
});
```

### 7.2 Disaster Recovery Plan

```markdown
## RTO & RPO Targets

- **RTO (Recovery Time Objective):** 4 hours
- **RPO (Recovery Point Objective):** 1 hour

## Recovery Procedures

### Database Recovery
1. Identify latest valid snapshot
2. Restore RDS instance from snapshot
3. Update connection strings
4. Verify data integrity
5. Resume application

### Application Recovery
1. Switch to backup region
2. Deploy latest stable Docker image
3. Update DNS records
4. Verify functionality
5. Monitor closely

### Complete Disaster Recovery
1. Activate DR site in secondary region
2. Restore database from backup
3. Deploy application containers
4. Update Route 53 failover
5. Redirect traffic
6. Verify all systems operational
```

---

## 8. Performance Optimization

### 8.1 Performance Monitoring

```typescript
// src/middleware/performance.middleware.ts
import * as prometheus from 'prom-client';

// Metrics
const httpRequestDuration = new prometheus.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.1, 0.5, 1, 2, 5]
});

const httpRequestTotal = new prometheus.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status']
});

const databaseQueryDuration = new prometheus.Histogram({
  name: 'database_query_duration_seconds',
  help: 'Duration of database queries',
  labelNames: ['operation', 'table'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1]
});

export function performanceMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
) {
  const start = Date.now();

  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    
    httpRequestDuration
      .labels(req.method, req.route?.path || req.path, res.statusCode.toString())
      .observe(duration);
    
    httpRequestTotal
      .labels(req.method, req.route?.path || req.path, res.statusCode.toString())
      .inc();
  });

  next();
}
```

### 8.2 Caching Strategy

```typescript
// src/services/cache/cache.service.ts
export class CacheService {
  // Multi-layer caching
  async get<T>(key: string): Promise<T | null> {
    // Layer 1: In-memory cache
    const memoryCache = this.memoryCache.get(key);
    if (memoryCache) return memoryCache;

    // Layer 2: Redis cache
    const redisCache = await redis.get(key);
    if (redisCache) {
      const parsed = JSON.parse(redisCache);
      this.memoryCache.set(key, parsed);
      return parsed;
    }

    return null;
  }

  // Cache with TTL
  async set(key: string, value: any, ttl: number = 3600): Promise<void> {
    // Store in Redis
    await redis.setex(key, ttl, JSON.stringify(value));
    
    // Store in memory (shorter TTL)
    this.memoryCache.set(key, value, Math.min(ttl, 300));
  }

  // Cache warming
  async warmCache(): Promise<void> {
    // Preload frequently accessed data
    const products = await prisma.product.findMany({
      where: { featured: true },
      take: 100
    });

    for (const product of products) {
      await this.set(`product:${product.id}`, product, 3600);
    }
  }
}
```

---

## Conclusion

This DevOps & Deployment guide provides a comprehensive, production-ready infrastructure setup with:

✅ **Automated CI/CD pipelines**  
✅ **Infrastructure as Code with Terraform**  
✅ **Container orchestration with ECS**  
✅ **Comprehensive monitoring & logging**  
✅ **Multiple deployment strategies**  
✅ **Disaster recovery procedures**  
✅ **Performance optimization techniques**

---

**Document Status:** ✅ Complete  
**Last Updated:** March 26, 2026  
**DevOps Team:** devops@company.com
