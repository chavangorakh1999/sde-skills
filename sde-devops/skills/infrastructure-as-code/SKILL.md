---
name: infrastructure-as-code
description: "Infrastructure as Code with Terraform: resource definitions, state management, modules, workspaces, and AWS/GCP patterns. Use when provisioning or modifying cloud infrastructure."
---

## Infrastructure as Code with Terraform

### Context

Infrastructure to provision or IaC problem: **$ARGUMENTS**

---

### Terraform Fundamentals

```hcl
# main.tf — providers and versions
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # allow 5.x, block 6.x
    }
  }

  # Remote state (NEVER use local state for team projects)
  backend "s3" {
    bucket         = "my-org-terraform-state"
    key            = "production/main.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"  # prevent concurrent applies
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "terraform"
      Project     = var.project_name
    }
  }
}
```

---

### Variables and Outputs

```hcl
# variables.tf
variable "environment" {
  description = "Deployment environment"
  type        = string
  validation {
    condition     = contains(["development", "staging", "production"], var.environment)
    error_message = "Environment must be development, staging, or production."
  }
}

variable "instance_type" {
  description = "EC2 instance type for the application"
  type        = string
  default     = "t3.medium"
}

variable "db_password" {
  description = "RDS database password"
  type        = string
  sensitive   = true  # redacted in output and state display
}

# outputs.tf
output "api_endpoint" {
  description = "Application Load Balancer DNS name"
  value       = aws_lb.main.dns_name
}

output "rds_endpoint" {
  description = "RDS instance endpoint"
  value       = aws_db_instance.main.endpoint
  sensitive   = true  # don't show in CI output
}
```

---

### Node.js App on AWS (ECS + RDS + Redis)

```hcl
# ecs.tf — ECS Fargate cluster and service
resource "aws_ecs_cluster" "main" {
  name = "${var.project_name}-${var.environment}"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

resource "aws_ecs_task_definition" "api" {
  family                   = "${var.project_name}-api"
  cpu                      = 512     # 0.5 vCPU
  memory                   = 1024    # 1 GB
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  execution_role_arn       = aws_iam_role.ecs_task_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([{
    name  = "api"
    image = "${var.ecr_image_uri}:${var.image_tag}"

    portMappings = [{
      containerPort = 3000
      protocol      = "tcp"
    }]

    environment = [
      { name = "NODE_ENV",   value = "production" },
      { name = "PORT",       value = "3000" },
    ]

    secrets = [
      # Pull secrets from SSM Parameter Store
      { name = "JWT_ACCESS_SECRET",  valueFrom = aws_ssm_parameter.jwt_access_secret.arn },
      { name = "MONGODB_URI",        valueFrom = aws_ssm_parameter.mongodb_uri.arn },
    ]

    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = aws_cloudwatch_log_group.api.name
        "awslogs-region"        = var.aws_region
        "awslogs-stream-prefix" = "api"
      }
    }

    healthCheck = {
      command     = ["CMD-SHELL", "wget -qO- http://localhost:3000/health || exit 1"]
      interval    = 30
      timeout     = 10
      retries     = 3
      startPeriod = 10
    }
  }])
}

resource "aws_ecs_service" "api" {
  name            = "api"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.api.arn
  desired_count   = 2

  launch_type = "FARGATE"

  network_configuration {
    subnets          = aws_subnet.private[*].id
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.api.arn
    container_name   = "api"
    container_port   = 3000
  }

  # Zero-downtime deployments
  deployment_minimum_healthy_percent = 100
  deployment_maximum_percent         = 200

  depends_on = [aws_lb_listener.https]
}
```

---

### Modules for Reusability

```hcl
# modules/rds/main.tf — reusable RDS module
variable "identifier" {}
variable "instance_class" { default = "db.t3.medium" }
variable "db_name" {}
variable "db_username" {}
variable "db_password" { sensitive = true }
variable "vpc_id" {}
variable "subnet_ids" { type = list(string) }

resource "aws_db_instance" "this" {
  identifier             = var.identifier
  engine                 = "postgres"
  engine_version         = "15.4"
  instance_class         = var.instance_class
  db_name                = var.db_name
  username               = var.db_username
  password               = var.db_password
  db_subnet_group_name   = aws_db_subnet_group.this.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  # Production settings
  multi_az               = true
  backup_retention_period = 7
  deletion_protection    = true
  storage_encrypted      = true

  skip_final_snapshot = false
  final_snapshot_identifier = "${var.identifier}-final"
}

output "endpoint"  { value = aws_db_instance.this.endpoint }
output "db_name"   { value = aws_db_instance.this.db_name }

# Usage in root module:
module "database" {
  source         = "./modules/rds"
  identifier     = "${var.project_name}-${var.environment}"
  db_name        = var.project_name
  db_username    = "appuser"
  db_password    = var.db_password
  vpc_id         = module.vpc.vpc_id
  subnet_ids     = module.vpc.private_subnets
}
```

---

### Workspaces for Environments

```bash
# Use workspaces to manage multiple environments with one codebase
terraform workspace new production
terraform workspace new staging
terraform workspace list

# Select workspace
terraform workspace select production

# Reference in config:
locals {
  environment = terraform.workspace
  is_production = terraform.workspace == "production"
}

resource "aws_ecs_service" "api" {
  desired_count = local.is_production ? 3 : 1
  # ...
}
```

---

### Terraform Workflow (GitOps)

```bash
# Development workflow
terraform init           # initialize providers and backend
terraform fmt            # format all .tf files
terraform validate       # check syntax
terraform plan -out=tfplan  # preview changes (save plan)
terraform show tfplan    # inspect plan
terraform apply tfplan   # apply saved plan (no interactive prompt)

# CI/CD workflow
terraform init -backend-config="key=${ENVIRONMENT}/main.tfstate"
terraform validate
terraform plan -out=tfplan -var="environment=${ENVIRONMENT}" -var="image_tag=${GIT_SHA}"
# Store tfplan artifact, require approval in PR
terraform apply tfplan

# Destroy (careful!)
terraform plan -destroy   # preview what would be destroyed
terraform destroy         # actually destroy (prompt)
```

---

### State Management

```bash
# View current state
terraform state list
terraform state show aws_ecs_service.api

# Import existing resource into state
terraform import aws_s3_bucket.assets my-existing-bucket-name

# Move resource in state (after refactoring)
terraform state mv aws_ecs_service.api module.ecs.aws_ecs_service.api

# Remove from state without destroying (hand off to another team)
terraform state rm aws_s3_bucket.logs

# Backup state before risky operations
terraform state pull > state-backup.json
```
