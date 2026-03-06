---
description: "Write Terraform infrastructure code for cloud resources: compute, networking, databases, and IAM"
argument-hint: "[infrastructure to provision, e.g. 'ECS Fargate service with RDS PostgreSQL and Redis on AWS']"
---

## Write Infrastructure as Code

Generate production-ready Terraform code for the specified infrastructure.

### Phase 1 — Architecture Design

Define the infrastructure components:
- **Compute**: ECS Fargate, EC2 Auto Scaling Group, Lambda, GKE, etc.
- **Networking**: VPC, subnets (public/private), NAT Gateway, security groups
- **Database**: RDS (multi-AZ), MongoDB Atlas, DynamoDB
- **Cache**: ElastiCache Redis, Memcached
- **Storage**: S3 buckets, EFS
- **Load Balancing**: ALB with target groups, listeners, rules
- **DNS**: Route53 records

Draw a text architecture diagram showing component relationships.

### Phase 2 — Terraform Structure

Organize files following best practices:
```
infrastructure/
  main.tf          — providers, backend config
  variables.tf     — all input variables with descriptions
  outputs.tf       — useful outputs (endpoints, ARNs)
  locals.tf        — computed values, naming conventions
  networking.tf    — VPC, subnets, security groups
  compute.tf       — ECS/EC2/Lambda resources
  database.tf      — RDS, Redis
  iam.tf           — IAM roles and policies
  monitoring.tf    — CloudWatch, alarms
  modules/         — reusable sub-modules
```

### Phase 3 — Networking (VPC)

Provision secure networking:
- VPC with appropriate CIDR block
- Public subnets (for ALB, NAT Gateway)
- Private subnets (for compute and databases)
- Internet Gateway for public subnets
- NAT Gateway per AZ for HA (or single NAT for cost savings)
- Route tables
- Security groups with least-privilege ingress/egress rules

### Phase 4 — Compute Resources

Write compute Terraform:
- Task definitions / launch templates with resource sizing
- Health checks matching application /health endpoint
- Auto scaling: target tracking (CPU 70%) or step scaling
- Rolling deployment configuration (0% unavailable during update)
- Log groups with retention policy

### Phase 5 — Database and Cache

Provision data layer:
- RDS: Multi-AZ for production, automated backups, deletion protection
- Encryption at rest with KMS
- Parameter groups for DB tuning
- Subnet groups in private subnets
- Security groups: only allow from application security group
- ElastiCache: cluster mode, auth token, TLS

### Phase 6 — IAM (Least Privilege)

Write IAM roles and policies:
- Task execution role: pull images, write logs, fetch SSM parameters
- Task role: only the AWS APIs the app actually needs
- S3 bucket policies: restrict to specific role and operations
- No wildcard resources in production policies

### Phase 7 — Variables and Environments

Make the code reusable across environments:
- All environment-specific values as variables
- Workspace-based conditional sizing (prod: multi-AZ, staging: single)
- .tfvars files for each environment (not committed if they contain secrets)
- Remote state with per-environment key

### Output

Provide:
1. **Architecture diagram** (text-based) with component relationships
2. **Complete Terraform files** for all phases above
3. **terraform.tfvars.example** with all required variables
4. **Backend configuration** for remote state
5. **Apply sequence** — any resources that must be applied before others
6. **Cost estimate** — approximate monthly cost for the provisioned resources
7. **Destroy safety check** — resources with deletion_protection, what to do before destroy
