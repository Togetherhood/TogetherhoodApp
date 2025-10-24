# AWS Infrastructure Setup Guide

## Overview

This guide covers setting up the complete AWS infrastructure for the Togetherhood monorepo, including:
- Amazon ECR (Elastic Container Registry)
- AWS App Runner services (Staging + Production)
- Amazon RDS MySQL databases (Staging + Production)
- AWS Secrets Manager for configuration
- IAM roles and policies
- CloudWatch monitoring

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│  GitHub Actions (Trunk-Based)                                    │
│    └─ main branch → Build → ECR (prod tag)                      │
│       (staging uses manual deployment or same artifact)          │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│  Amazon ECR                                                      │
│    Repository: togetherhood-app                                 │
│    ├─ staging (latest staging build)                            │
│    ├─ staging-{sha} (specific builds)                           │
│    ├─ prod (latest production build)                            │
│    └─ prod-{sha} (specific builds)                              │
└──────────────────┬─────────────────┬────────────────────────────┘
                   │                 │
                   ↓                 ↓
┌──────────────────────────┐  ┌──────────────────────────┐
│  App Runner - Staging    │  │  App Runner - Production │
│    - 0.5 vCPU / 1 GB     │  │    - 1 vCPU / 2 GB       │
│    - Auto-deploys on     │  │    - Auto-deploys on     │
│      'staging' tag       │  │      'prod' tag          │
│    - staging.domain.com  │  │    - app.domain.com      │
└──────────┬───────────────┘  └──────────┬───────────────┘
           │                              │
           ↓                              ↓
┌──────────────────────────┐  ┌──────────────────────────┐
│  RDS MySQL - Staging     │  │  RDS MySQL - Production  │
│    - db.t3.micro         │  │    - db.t3.large         │
│    - 20 GB storage       │  │    - 100 GB storage      │
└──────────────────────────┘  └──────────────────────────┘
           │                              │
           ↓                              ↓
┌──────────────────────────┐  ┌──────────────────────────┐
│  Secrets Manager         │  │  Secrets Manager         │
│    staging/config        │  │    production/config     │
└──────────────────────────┘  └──────────────────────────┘
```

---

## Prerequisites

- [ ] AWS Account with admin access
- [ ] AWS CLI installed and configured (`aws configure`)
- [ ] jq installed (for JSON parsing)
- [ ] Domain name (optional, for custom domains)

---

## Setup Overview

| Step | Component | Estimated Time |
|------|-----------|----------------|
| 1 | ECR Repository | 5 minutes |
| 2 | IAM Roles & Policies | 10 minutes |
| 3 | RDS Databases | 15 minutes |
| 4 | Secrets Manager | 10 minutes |
| 5 | App Runner Services | 15 minutes |
| 6 | Monitoring & Alarms | 10 minutes |
| **Total** | | **~65 minutes** |

---

## Step 1: Create ECR Repository

### Create Repository

```bash
# Set variables
export AWS_REGION="us-east-1"
export ECR_REPO_NAME="togetherhood-app"

# Create ECR repository
aws ecr create-repository \
    --repository-name ${ECR_REPO_NAME} \
    --region ${AWS_REGION} \
    --image-scanning-configuration scanOnPush=true \
    --encryption-configuration encryptionType=AES256 \
    --tags Key=Application,Value=TogetherhoodApp

# Get repository URI (save this for GitHub Actions)
export ECR_REPO_URI=$(aws ecr describe-repositories \
    --repository-names ${ECR_REPO_NAME} \
    --region ${AWS_REGION} \
    --query 'repositories[0].repositoryUri' \
    --output text)

echo "✅ ECR Repository created: ${ECR_REPO_URI}"
```

### Configure Lifecycle Policy

Create `aws-config/ecr-lifecycle-policy.json`:

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 10 staging images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["staging"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "Keep last 10 production images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["prod"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 3,
      "description": "Expire untagged images after 7 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": { "type": "expire" }
    }
  ]
}
```

Apply lifecycle policy:

```bash
aws ecr put-lifecycle-policy \
    --repository-name ${ECR_REPO_NAME} \
    --lifecycle-policy-text file://aws-config/ecr-lifecycle-policy.json \
    --region ${AWS_REGION}

echo "✅ Lifecycle policy applied"
```

---

## Step 2: Create IAM Roles & Policies

### ECR Access Role (for App Runner to pull images)

```bash
# Create trust policy
cat > aws-config/ecr-access-role-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "build.apprunner.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

# Create role
aws iam create-role \
    --role-name TogetherhoodAppRunnerECRAccessRole \
    --assume-role-policy-document file://aws-config/ecr-access-role-trust-policy.json

# Attach AWS managed policy for ECR access
aws iam attach-role-policy \
    --role-name TogetherhoodAppRunnerECRAccessRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSAppRunnerServicePolicyForECRAccess

# Get role ARN (save for later)
export ECR_ACCESS_ROLE_ARN=$(aws iam get-role \
    --role-name TogetherhoodAppRunnerECRAccessRole \
    --query 'Role.Arn' \
    --output text)

echo "✅ ECR Access Role created: ${ECR_ACCESS_ROLE_ARN}"
```

### Instance Roles (for App Runner services to access AWS resources)

Get AWS Account ID:

```bash
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

#### Staging Instance Role

```bash
# Create trust policy for instance role
cat > aws-config/instance-role-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "tasks.apprunner.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

# Create staging instance role
aws iam create-role \
    --role-name TogetherhoodAppRunnerInstanceRoleStaging \
    --assume-role-policy-document file://aws-config/instance-role-trust-policy.json

# Create staging policy
cat > aws-config/instance-policy-staging.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": [
        "arn:aws:secretsmanager:${AWS_REGION}:${AWS_ACCOUNT_ID}:secret:togetherhood-app/staging/config*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::togetherhood-staging-uploads",
        "arn:aws:s3:::togetherhood-staging-uploads/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": ["arn:aws:kms:${AWS_REGION}:${AWS_ACCOUNT_ID}:key/*"],
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "secretsmanager.${AWS_REGION}.amazonaws.com"
        }
      }
    }
  ]
}
EOF

# Create and attach staging policy
aws iam create-policy \
    --policy-name TogetherhoodAppRunnerInstancePolicyStaging \
    --policy-document file://aws-config/instance-policy-staging.json

export STAGING_POLICY_ARN=$(aws iam list-policies \
    --query "Policies[?PolicyName=='TogetherhoodAppRunnerInstancePolicyStaging'].Arn" \
    --output text)

aws iam attach-role-policy \
    --role-name TogetherhoodAppRunnerInstanceRoleStaging \
    --policy-arn ${STAGING_POLICY_ARN}

# Get staging instance role ARN
export STAGING_INSTANCE_ROLE_ARN=$(aws iam get-role \
    --role-name TogetherhoodAppRunnerInstanceRoleStaging \
    --query 'Role.Arn' \
    --output text)

echo "✅ Staging Instance Role created: ${STAGING_INSTANCE_ROLE_ARN}"
```

#### Production Instance Role

```bash
# Create production instance role
aws iam create-role \
    --role-name TogetherhoodAppRunnerInstanceRoleProduction \
    --assume-role-policy-document file://aws-config/instance-role-trust-policy.json

# Create production policy
cat > aws-config/instance-policy-production.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": [
        "arn:aws:secretsmanager:${AWS_REGION}:${AWS_ACCOUNT_ID}:secret:togetherhood-app/production/config*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::togetherhood-production-uploads",
        "arn:aws:s3:::togetherhood-production-uploads/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": ["arn:aws:kms:${AWS_REGION}:${AWS_ACCOUNT_ID}:key/*"],
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "secretsmanager.${AWS_REGION}.amazonaws.com"
        }
      }
    }
  ]
}
EOF

# Create and attach production policy
aws iam create-policy \
    --policy-name TogetherhoodAppRunnerInstancePolicyProduction \
    --policy-document file://aws-config/instance-policy-production.json

export PROD_POLICY_ARN=$(aws iam list-policies \
    --query "Policies[?PolicyName=='TogetherhoodAppRunnerInstancePolicyProduction'].Arn" \
    --output text)

aws iam attach-role-policy \
    --role-name TogetherhoodAppRunnerInstanceRoleProduction \
    --policy-arn ${PROD_POLICY_ARN}

# Get production instance role ARN
export PROD_INSTANCE_ROLE_ARN=$(aws iam get-role \
    --role-name TogetherhoodAppRunnerInstanceRoleProduction \
    --query 'Role.Arn' \
    --output text)

echo "✅ Production Instance Role created: ${PROD_INSTANCE_ROLE_ARN}"
```

---

## Step 3: Create RDS Databases

### Create VPC Security Group for RDS

```bash
# Get default VPC ID
export VPC_ID=$(aws ec2 describe-vpcs \
    --filters "Name=isDefault,Values=true" \
    --query 'Vpcs[0].VpcId' \
    --output text)

# Create security group
aws ec2 create-security-group \
    --group-name togetherhood-rds-sg \
    --description "Security group for Togetherhood RDS instances" \
    --vpc-id ${VPC_ID}

export RDS_SG_ID=$(aws ec2 describe-security-groups \
    --filters "Name=group-name,Values=togetherhood-rds-sg" \
    --query 'SecurityGroups[0].GroupId' \
    --output text)

# Allow MySQL access from App Runner (will be updated with App Runner security group later)
aws ec2 authorize-security-group-ingress \
    --group-id ${RDS_SG_ID} \
    --protocol tcp \
    --port 3306 \
    --cidr 0.0.0.0/0  # TEMP: Will restrict after App Runner creation

echo "✅ RDS Security Group created: ${RDS_SG_ID}"
```

### Create Staging Database

```bash
# Create staging RDS instance
aws rds create-db-instance \
    --db-instance-identifier togetherhood-staging-db \
    --db-instance-class db.t3.micro \
    --engine mysql \
    --engine-version 8.0.35 \
    --master-username admin \
    --master-user-password "$(openssl rand -base64 32)" \
    --allocated-storage 20 \
    --storage-type gp3 \
    --db-name togetherhood_staging \
    --backup-retention-period 7 \
    --preferred-backup-window "03:00-04:00" \
    --preferred-maintenance-window "mon:04:00-mon:05:00" \
    --vpc-security-group-ids ${RDS_SG_ID} \
    --publicly-accessible false \
    --storage-encrypted \
    --enable-cloudwatch-logs-exports '["error","general","slowquery"]' \
    --tags Key=Environment,Value=Staging Key=Application,Value=TogetherhoodApp \
    --region ${AWS_REGION}

echo "⏳ Waiting for staging database to be available (5-10 minutes)..."
aws rds wait db-instance-available \
    --db-instance-identifier togetherhood-staging-db \
    --region ${AWS_REGION}

# Get staging DB endpoint
export STAGING_DB_ENDPOINT=$(aws rds describe-db-instances \
    --db-instance-identifier togetherhood-staging-db \
    --region ${AWS_REGION} \
    --query 'DBInstances[0].Endpoint.Address' \
    --output text)

echo "✅ Staging Database created: ${STAGING_DB_ENDPOINT}"
```

### Create Production Database

```bash
# Create production RDS instance
aws rds create-db-instance \
    --db-instance-identifier togetherhood-production-db \
    --db-instance-class db.t3.large \
    --engine mysql \
    --engine-version 8.0.35 \
    --master-username admin \
    --master-user-password "$(openssl rand -base64 32)" \
    --allocated-storage 100 \
    --storage-type gp3 \
    --iops 3000 \
    --db-name togetherhood_production \
    --backup-retention-period 30 \
    --preferred-backup-window "03:00-04:00" \
    --preferred-maintenance-window "mon:04:00-mon:05:00" \
    --vpc-security-group-ids ${RDS_SG_ID} \
    --publicly-accessible false \
    --storage-encrypted \
    --enable-cloudwatch-logs-exports '["error","general","slowquery"]' \
    --multi-az \
    --tags Key=Environment,Value=Production Key=Application,Value=TogetherhoodApp \
    --region ${AWS_REGION}

echo "⏳ Waiting for production database to be available (10-15 minutes)..."
aws rds wait db-instance-available \
    --db-instance-identifier togetherhood-production-db \
    --region ${AWS_REGION}

# Get production DB endpoint
export PROD_DB_ENDPOINT=$(aws rds describe-db-instances \
    --db-instance-identifier togetherhood-production-db \
    --region ${AWS_REGION} \
    --query 'DBInstances[0].Endpoint.Address' \
    --output text)

echo "✅ Production Database created: ${PROD_DB_ENDPOINT}"
```

**Important:** Save the master passwords somewhere secure (e.g., 1Password, AWS Secrets Manager).

---

## Step 4: Configure Secrets Manager

See **[AWS Secrets Manager Integration Guide](aws-secrets-manager.md)** for detailed instructions.

Quick setup:

```bash
# Create staging secrets
aws secretsmanager create-secret \
    --name togetherhood-app/staging/config \
    --description "Staging configuration" \
    --secret-string '{
      "ConnectionStrings__DefaultConnection": "Server='${STAGING_DB_ENDPOINT}';Database=togetherhood_staging;User=admin;Password=<STAGING_DB_PASSWORD>",
      "JwtSettings__Secret": "'$(openssl rand -base64 64)'",
      "SendGrid__ApiKey": "SG.sandbox_key",
      "Twilio__AccountSid": "test_sid"
    }' \
    --region ${AWS_REGION}

# Create production secrets
aws secretsmanager create-secret \
    --name togetherhood-app/production/config \
    --description "Production configuration" \
    --secret-string '{
      "ConnectionStrings__DefaultConnection": "Server='${PROD_DB_ENDPOINT}';Database=togetherhood_production;User=admin;Password=<PROD_DB_PASSWORD>",
      "JwtSettings__Secret": "'$(openssl rand -base64 64)'",
      "SendGrid__ApiKey": "<REAL_SENDGRID_KEY>",
      "Twilio__AccountSid": "<REAL_TWILIO_SID>"
    }' \
    --region ${AWS_REGION}

echo "✅ Secrets created in Secrets Manager"
```

---

## Step 5: Create App Runner Services

### Create Staging Service

**Note:** You need to push an initial Docker image to ECR with tag `staging` before creating the service.

```bash
# Staging service configuration
cat > aws-config/apprunner-staging.json <<EOF
{
  "ServiceName": "togetherhood-app-staging",
  "SourceConfiguration": {
    "ImageRepository": {
      "ImageIdentifier": "${ECR_REPO_URI}:staging",
      "ImageRepositoryType": "ECR",
      "ImageConfiguration": {
        "Port": "8080",
        "RuntimeEnvironmentVariables": {
          "ASPNETCORE_ENVIRONMENT": "Staging"
        },
        "RuntimeEnvironmentSecrets": {
          "ConnectionStrings__DefaultConnection": "arn:aws:secretsmanager:${AWS_REGION}:${AWS_ACCOUNT_ID}:secret:togetherhood-app/staging/config:ConnectionStrings__DefaultConnection::",
          "JwtSettings__Secret": "arn:aws:secretsmanager:${AWS_REGION}:${AWS_ACCOUNT_ID}:secret:togetherhood-app/staging/config:JwtSettings__Secret::"
        }
      }
    },
    "AuthenticationConfiguration": {
      "AccessRoleArn": "${ECR_ACCESS_ROLE_ARN}"
    },
    "AutoDeploymentsEnabled": true
  },
  "InstanceConfiguration": {
    "Cpu": "0.5 vCPU",
    "Memory": "1 GB",
    "InstanceRoleArn": "${STAGING_INSTANCE_ROLE_ARN}"
  },
  "HealthCheckConfiguration": {
    "Protocol": "HTTP",
    "Path": "/api/health",
    "Interval": 10,
    "Timeout": 5,
    "HealthyThreshold": 1,
    "UnhealthyThreshold": 5
  }
}
EOF

# Create staging service
aws apprunner create-service \
    --cli-input-json file://aws-config/apprunner-staging.json \
    --region ${AWS_REGION}

echo "⏳ Waiting for staging service to be created (3-5 minutes)..."

# Get staging service details
export STAGING_SERVICE_ARN=$(aws apprunner list-services \
    --region ${AWS_REGION} \
    --query "ServiceSummaryList[?ServiceName=='togetherhood-app-staging'].ServiceArn" \
    --output text)

export STAGING_SERVICE_URL=$(aws apprunner list-services \
    --region ${AWS_REGION} \
    --query "ServiceSummaryList[?ServiceName=='togetherhood-app-staging'].ServiceUrl" \
    --output text)

echo "✅ Staging Service created"
echo "   ARN: ${STAGING_SERVICE_ARN}"
echo "   URL: https://${STAGING_SERVICE_URL}"
```

### Create Production Service

```bash
# Production service configuration
cat > aws-config/apprunner-production.json <<EOF
{
  "ServiceName": "togetherhood-app-production",
  "SourceConfiguration": {
    "ImageRepository": {
      "ImageIdentifier": "${ECR_REPO_URI}:prod",
      "ImageRepositoryType": "ECR",
      "ImageConfiguration": {
        "Port": "8080",
        "RuntimeEnvironmentVariables": {
          "ASPNETCORE_ENVIRONMENT": "Production"
        },
        "RuntimeEnvironmentSecrets": {
          "ConnectionStrings__DefaultConnection": "arn:aws:secretsmanager:${AWS_REGION}:${AWS_ACCOUNT_ID}:secret:togetherhood-app/production/config:ConnectionStrings__DefaultConnection::",
          "JwtSettings__Secret": "arn:aws:secretsmanager:${AWS_REGION}:${AWS_ACCOUNT_ID}:secret:togetherhood-app/production/config:JwtSettings__Secret::"
        }
      }
    },
    "AuthenticationConfiguration": {
      "AccessRoleArn": "${ECR_ACCESS_ROLE_ARN}"
    },
    "AutoDeploymentsEnabled": true
  },
  "InstanceConfiguration": {
    "Cpu": "1 vCPU",
    "Memory": "2 GB",
    "InstanceRoleArn": "${PROD_INSTANCE_ROLE_ARN}"
  },
  "HealthCheckConfiguration": {
    "Protocol": "HTTP",
    "Path": "/api/health",
    "Interval": 10,
    "Timeout": 5,
    "HealthyThreshold": 1,
    "UnhealthyThreshold": 5
  }
}
EOF

# Create production service
aws apprunner create-service \
    --cli-input-json file://aws-config/apprunner-production.json \
    --region ${AWS_REGION}

echo "⏳ Waiting for production service to be created (3-5 minutes)..."

# Get production service details
export PROD_SERVICE_ARN=$(aws apprunner list-services \
    --region ${AWS_REGION} \
    --query "ServiceSummaryList[?ServiceName=='togetherhood-app-production'].ServiceArn" \
    --output text)

export PROD_SERVICE_URL=$(aws apprunner list-services \
    --region ${AWS_REGION} \
    --query "ServiceSummaryList[?ServiceName=='togetherhood-app-production'].ServiceUrl" \
    --output text)

echo "✅ Production Service created"
echo "   ARN: ${PROD_SERVICE_ARN}"
echo "   URL: https://${PROD_SERVICE_URL}"
```

---

## Step 6: Configure Monitoring & Alarms

### CloudWatch Dashboard

```bash
# Create dashboard JSON
cat > aws-config/cloudwatch-dashboard.json <<EOF
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/AppRunner", "Requests", {"stat": "Sum"}],
          [".", "RequestLatency", {"stat": "Average"}],
          [".", "ActiveInstances", {"stat": "Average"}]
        ],
        "period": 300,
        "stat": "Average",
        "region": "${AWS_REGION}",
        "title": "App Runner - Overview"
      }
    }
  ]
}
EOF

# Create dashboard
aws cloudwatch put-dashboard \
    --dashboard-name TogetherhoodApp \
    --dashboard-body file://aws-config/cloudwatch-dashboard.json

echo "✅ CloudWatch Dashboard created"
```

### CloudWatch Alarms

```bash
# Create SNS topic for alarms
aws sns create-topic --name togetherhood-alarms

export SNS_TOPIC_ARN=$(aws sns list-topics \
    --query "Topics[?contains(TopicArn, 'togetherhood-alarms')].TopicArn" \
    --output text)

# Subscribe your email
aws sns subscribe \
    --topic-arn ${SNS_TOPIC_ARN} \
    --protocol email \
    --notification-endpoint your-email@example.com

# Production health check alarm
aws cloudwatch put-metric-alarm \
    --alarm-name prod-health-check-failed \
    --alarm-description "Production health checks failing" \
    --metric-name 4xxStatusResponses \
    --namespace AWS/AppRunner \
    --statistic Sum \
    --period 300 \
    --threshold 10 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --alarm-actions ${SNS_TOPIC_ARN}

echo "✅ CloudWatch Alarms created"
```

---

## Summary & Export Variables

Save all important values:

```bash
# Create environment file for GitHub Actions
cat > aws-infrastructure-outputs.env <<EOF
# ECR
ECR_REPO_URI=${ECR_REPO_URI}

# IAM Roles
ECR_ACCESS_ROLE_ARN=${ECR_ACCESS_ROLE_ARN}
STAGING_INSTANCE_ROLE_ARN=${STAGING_INSTANCE_ROLE_ARN}
PROD_INSTANCE_ROLE_ARN=${PROD_INSTANCE_ROLE_ARN}

# RDS
STAGING_DB_ENDPOINT=${STAGING_DB_ENDPOINT}
PROD_DB_ENDPOINT=${PROD_DB_ENDPOINT}

# App Runner
STAGING_SERVICE_ARN=${STAGING_SERVICE_ARN}
STAGING_SERVICE_URL=${STAGING_SERVICE_URL}
PROD_SERVICE_ARN=${PROD_SERVICE_ARN}
PROD_SERVICE_URL=${PROD_SERVICE_URL}

# Other
AWS_REGION=${AWS_REGION}
AWS_ACCOUNT_ID=${AWS_ACCOUNT_ID}
EOF

echo "✅ All infrastructure created!"
echo ""
echo "📄 Variables saved to: aws-infrastructure-outputs.env"
echo ""
echo "🔐 Add these to GitHub Secrets:"
echo "  - AWS_ACCESS_KEY_ID"
echo "  - AWS_SECRET_ACCESS_KEY"
echo "  - STAGING_SERVICE_URL=${STAGING_SERVICE_URL}"
echo "  - PROD_SERVICE_URL=${PROD_SERVICE_URL}"
echo "  - STAGING_SERVICE_ARN=${STAGING_SERVICE_ARN}"
echo "  - PROD_SERVICE_ARN=${PROD_SERVICE_ARN}"
```

---

## Next Steps

1. ✅ Configure GitHub Secrets with the values above
2. ✅ Set up custom domains (optional) - see [Custom Domains](#custom-domains)
3. ✅ Push initial Docker images to ECR
4. ✅ Test deployments
5. ✅ Configure monitoring alerts

---

## Custom Domains (Optional)

### Add Custom Domain to App Runner

```bash
# Associate custom domain with production
aws apprunner associate-custom-domain \
    --service-arn ${PROD_SERVICE_ARN} \
    --domain-name app.togetherhood.com \
    --region ${AWS_REGION}

# Get validation records
aws apprunner describe-custom-domains \
    --service-arn ${PROD_SERVICE_ARN} \
    --region ${AWS_REGION}

# Add the CNAME records to your DNS provider
# (Record details will be in the output above)

# Associate staging domain
aws apprunner associate-custom-domain \
    --service-arn ${STAGING_SERVICE_ARN} \
    --domain-name staging.togetherhood.com \
    --region ${AWS_REGION}
```

---

## Cost Estimate

| Resource | Staging | Production | Monthly Total |
|----------|---------|------------|---------------|
| **App Runner** | $30 | $60 | $90 |
| **RDS MySQL** | $15 | $150 | $165 |
| **ECR Storage** | $5 | $5 | $10 |
| **Secrets Manager** | $1 | $1 | $2 |
| **Data Transfer** | $10 | $30 | $40 |
| **Backups** | $5 | $20 | $25 |
| **Total** | **~$66** | **~$266** | **~$332/month** |

---

## Cleanup (for testing only)

**WARNING:** This will delete everything!

```bash
# Delete App Runner services
aws apprunner delete-service --service-arn ${STAGING_SERVICE_ARN}
aws apprunner delete-service --service-arn ${PROD_SERVICE_ARN}

# Delete RDS instances
aws rds delete-db-instance --db-instance-identifier togetherhood-staging-db --skip-final-snapshot
aws rds delete-db-instance --db-instance-identifier togetherhood-production-db --skip-final-snapshot

# Delete secrets
aws secretsmanager delete-secret --secret-id togetherhood-app/staging/config --force-delete-without-recovery
aws secretsmanager delete-secret --secret-id togetherhood-app/production/config --force-delete-without-recovery

# Delete ECR repository
aws ecr delete-repository --repository-name togetherhood-app --force

# Delete IAM roles and policies (manually via console or CLI)
```

---

## Related Guides

- [AWS Secrets Manager Integration](aws-secrets-manager.md)
- [Staging Environment Setup](staging-environment.md)
- [CI/CD Testing Strategy](cicd-testing.md)
