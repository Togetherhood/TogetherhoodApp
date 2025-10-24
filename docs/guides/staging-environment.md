# Staging Environment Setup Guide

## Overview

This guide covers setting up a complete **staging environment** in AWS App Runner that mirrors production but deploys from the `develop` branch for testing and QA.

---

## Why Staging?

**Benefits:**
- ✅ Test changes in production-like environment before going live
- ✅ QA and stakeholder demos without affecting production
- ✅ Catch integration issues early
- ✅ Test database migrations safely
- ✅ Validate deployment process
- ✅ Run E2E tests against deployed environment

**Key Differences from Production:**
- Deploys from `develop` branch instead of `main`
- Uses separate RDS database (staging data)
- Uses separate Secrets Manager configuration
- Lower resource allocation (cheaper)
- May use test/sandbox external services (SendGrid, Twilio, etc.)

---

## Architecture

### Environment Flow

```
Developer pushes to 'develop' branch
    ↓
GitHub Actions: cd-staging.yml
    ↓
Build Docker image
    ↓
Push to ECR with tag 'staging'
    ↓
App Runner Staging Service auto-deploys
    ↓
Available at: https://staging-xxxxxx.us-east-1.awsapprunner.com
```

### Staging vs Production

| Aspect | Staging | Production |
|--------|---------|------------|
| **Git Branch** | `develop` | `main` |
| **ECR Tag** | `staging` | `prod` |
| **App Runner Service** | `togetherhood-app-staging` | `togetherhood-app-production` |
| **Database** | RDS Staging (db.t3.micro) | RDS Production (db.t3.large) |
| **Secrets** | `togetherhood-app/staging/config` | `togetherhood-app/production/config` |
| **CPU/Memory** | 0.5 vCPU / 1 GB | 1 vCPU / 2 GB |
| **Domain** | `staging.togetherhood.com` | `app.togetherhood.com` |
| **External Services** | Sandbox/Test (SendGrid, Twilio) | Production APIs |

---

## Setup Steps

### Step 1: Create Staging RDS Database (Optional)

If you want a separate staging database:

```bash
# Variables
AWS_REGION="us-east-1"
DB_INSTANCE_ID="togetherhood-staging-db"

# Create staging database
aws rds create-db-instance \
    --db-instance-identifier ${DB_INSTANCE_ID} \
    --db-instance-class db.t3.micro \
    --engine mysql \
    --engine-version 8.0.35 \
    --master-username admin \
    --master-user-password <STRONG_PASSWORD> \
    --allocated-storage 20 \
    --storage-type gp3 \
    --db-name togetherhood_staging \
    --backup-retention-period 7 \
    --preferred-backup-window "03:00-04:00" \
    --preferred-maintenance-window "mon:04:00-mon:05:00" \
    --vpc-security-group-ids <SECURITY_GROUP_ID> \
    --db-subnet-group-name <SUBNET_GROUP_NAME> \
    --publicly-accessible false \
    --storage-encrypted \
    --enable-cloudwatch-logs-exports '["error","general","slowquery"]' \
    --tags Key=Environment,Value=Staging Key=Application,Value=TogetherhoodApp \
    --region ${AWS_REGION}

# Wait for database to be available
aws rds wait db-instance-available \
    --db-instance-identifier ${DB_INSTANCE_ID} \
    --region ${AWS_REGION}

# Get connection endpoint
DB_ENDPOINT=$(aws rds describe-db-instances \
    --db-instance-identifier ${DB_INSTANCE_ID} \
    --region ${AWS_REGION} \
    --query 'DBInstances[0].Endpoint.Address' \
    --output text)

echo "Staging DB Endpoint: ${DB_ENDPOINT}"
```

**Note:** If you want to use the same RDS instance but different database, just create a new database in existing RDS:

```sql
CREATE DATABASE togetherhood_staging;
CREATE USER 'staging_user'@'%' IDENTIFIED BY 'strong_password';
GRANT ALL PRIVILEGES ON togetherhood_staging.* TO 'staging_user'@'%';
FLUSH PRIVILEGES;
```

### Step 2: Create Staging Secrets in Secrets Manager

**File: `aws-config/secrets-staging.json`:**

```json
{
  "ConnectionStrings__DefaultConnection": "Server=staging-db-endpoint.us-east-1.rds.amazonaws.com;Database=togetherhood_staging;User=staging_user;Password=STAGING_PASSWORD",
  "JwtSettings__Secret": "<different-from-prod-64-char-secret>",
  "JwtSettings__Issuer": "TogetherhoodAPI",
  "JwtSettings__Audience": "TogetherhoodWeb",
  "SendGrid__ApiKey": "<SENDGRID_SANDBOX_KEY>",
  "SendGrid__FromEmail": "staging@togetherhood.com",
  "Twilio__AccountSid": "<TWILIO_TEST_SID>",
  "Twilio__AuthToken": "<TWILIO_TEST_TOKEN>",
  "AWS__AccessKeyId": "<AWS_KEY_FOR_STAGING_S3>",
  "AWS__SecretAccessKey": "<AWS_SECRET_FOR_STAGING_S3>",
  "AWS__S3BucketName": "togetherhood-staging-uploads",
  "Tipalti__ApiKey": "<TIPALTI_SANDBOX_KEY>",
  "Tipalti__ApiUrl": "https://ui2.sandbox.tipalti.com/"
}
```

**Important:** Use **test/sandbox** credentials for external services in staging!

Create the secret:

```bash
aws secretsmanager create-secret \
    --name togetherhood-app/staging/config \
    --description "Staging configuration for Togetherhood App" \
    --secret-string file://aws-config/secrets-staging.json \
    --tags Key=Environment,Value=Staging Key=Application,Value=TogetherhoodApp \
    --region us-east-1

# Verify secret was created
aws secretsmanager describe-secret \
    --secret-id togetherhood-app/staging/config \
    --region us-east-1
```

### Step 3: Create IAM Roles for Staging App Runner

These are similar to production but for the staging service:

```bash
# Use the same ECR access role as production (already created)
# But create a separate instance role for staging

# Create instance role for staging
aws iam create-role \
    --role-name TogetherhoodAppRunnerInstanceRoleStaging \
    --assume-role-policy-document '{
      "Version": "2012-10-17",
      "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "tasks.apprunner.amazonaws.com"},
        "Action": "sts:AssumeRole"
      }]
    }'

# Create staging-specific policy (same as prod but scoped to staging secrets)
cat > aws-config/app-runner-staging-secrets-policy.json <<EOF
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
        "arn:aws:secretsmanager:us-east-1:${AWS_ACCOUNT_ID}:secret:togetherhood-app/staging/config*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::togetherhood-staging-uploads/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": ["arn:aws:kms:us-east-1:${AWS_ACCOUNT_ID}:key/*"],
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "secretsmanager.us-east-1.amazonaws.com"
        }
      }
    }
  ]
}
EOF

# Create policy
aws iam create-policy \
    --policy-name TogetherhoodAppRunnerStagingPolicy \
    --policy-document file://aws-config/app-runner-staging-secrets-policy.json

# Attach policy to staging instance role
STAGING_POLICY_ARN=$(aws iam list-policies \
    --query "Policies[?PolicyName=='TogetherhoodAppRunnerStagingPolicy'].Arn" \
    --output text)

aws iam attach-role-policy \
    --role-name TogetherhoodAppRunnerInstanceRoleStaging \
    --policy-arn ${STAGING_POLICY_ARN}

# Get staging instance role ARN
STAGING_INSTANCE_ROLE_ARN=$(aws iam get-role \
    --role-name TogetherhoodAppRunnerInstanceRoleStaging \
    --query 'Role.Arn' \
    --output text)

echo "Staging Instance Role ARN: ${STAGING_INSTANCE_ROLE_ARN}"
```

### Step 4: Create Staging App Runner Service

**File: `aws-config/apprunner-staging.json`:**

```json
{
  "ServiceName": "togetherhood-app-staging",
  "SourceConfiguration": {
    "ImageRepository": {
      "ImageIdentifier": "<ECR_REPO_URI>:staging",
      "ImageRepositoryType": "ECR",
      "ImageConfiguration": {
        "Port": "8080",
        "RuntimeEnvironmentVariables": {
          "ASPNETCORE_ENVIRONMENT": "Staging"
        },
        "RuntimeEnvironmentSecrets": {
          "ConnectionStrings__DefaultConnection": "arn:aws:secretsmanager:us-east-1:<ACCOUNT_ID>:secret:togetherhood-app/staging/config:ConnectionStrings__DefaultConnection::",
          "JwtSettings__Secret": "arn:aws:secretsmanager:us-east-1:<ACCOUNT_ID>:secret:togetherhood-app/staging/config:JwtSettings__Secret::",
          "SendGrid__ApiKey": "arn:aws:secretsmanager:us-east-1:<ACCOUNT_ID>:secret:togetherhood-app/staging/config:SendGrid__ApiKey::",
          "Twilio__AccountSid": "arn:aws:secretsmanager:us-east-1:<ACCOUNT_ID>:secret:togetherhood-app/staging/config:Twilio__AccountSid::",
          "Twilio__AuthToken": "arn:aws:secretsmanager:us-east-1:<ACCOUNT_ID>:secret:togetherhood-app/staging/config:Twilio__AuthToken::"
        }
      }
    },
    "AuthenticationConfiguration": {
      "AccessRoleArn": "<ECR_ACCESS_ROLE_ARN>"
    },
    "AutoDeploymentsEnabled": true
  },
  "InstanceConfiguration": {
    "Cpu": "0.5 vCPU",
    "Memory": "1 GB",
    "InstanceRoleArn": "<STAGING_INSTANCE_ROLE_ARN>"
  },
  "HealthCheckConfiguration": {
    "Protocol": "HTTP",
    "Path": "/api/health",
    "Interval": 10,
    "Timeout": 5,
    "HealthyThreshold": 1,
    "UnhealthyThreshold": 5
  },
  "Tags": [
    {
      "Key": "Environment",
      "Value": "Staging"
    },
    {
      "Key": "Application",
      "Value": "TogetherhoodApp"
    }
  ]
}
```

Replace placeholders and create service:

```bash
# Get ECR repository URI (from production setup)
ECR_REPO_URI=$(aws ecr describe-repositories \
    --repository-names togetherhood-app \
    --region us-east-1 \
    --query 'repositories[0].repositoryUri' \
    --output text)

# Get ECR access role ARN (from production setup)
ECR_ACCESS_ROLE_ARN=$(aws iam get-role \
    --role-name TogetherhoodAppRunnerECRAccessRole \
    --query 'Role.Arn' \
    --output text)

# Get account ID
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Replace placeholders in config
sed -i "s|<ECR_REPO_URI>|${ECR_REPO_URI}|g" aws-config/apprunner-staging.json
sed -i "s|<ACCOUNT_ID>|${AWS_ACCOUNT_ID}|g" aws-config/apprunner-staging.json
sed -i "s|<ECR_ACCESS_ROLE_ARN>|${ECR_ACCESS_ROLE_ARN}|g" aws-config/apprunner-staging.json
sed -i "s|<STAGING_INSTANCE_ROLE_ARN>|${STAGING_INSTANCE_ROLE_ARN}|g" aws-config/apprunner-staging.json

# Create staging App Runner service
aws apprunner create-service \
    --cli-input-json file://aws-config/apprunner-staging.json \
    --region us-east-1

# Wait for service to be created
echo "Waiting for staging service to be created... (this takes 3-5 minutes)"

# Get staging service ARN and URL
STAGING_SERVICE_ARN=$(aws apprunner list-services \
    --region us-east-1 \
    --query "ServiceSummaryList[?ServiceName=='togetherhood-app-staging'].ServiceArn" \
    --output text)

STAGING_SERVICE_URL=$(aws apprunner list-services \
    --region us-east-1 \
    --query "ServiceSummaryList[?ServiceName=='togetherhood-app-staging'].ServiceUrl" \
    --output text)

echo "Staging Service ARN: ${STAGING_SERVICE_ARN}"
echo "Staging Service URL: https://${STAGING_SERVICE_URL}"
```

### Step 4b: Configure Custom Domain (Optional)

If you want to use a custom domain like `beta.togetherhood.us` for staging:

**See:** **[DNS & Custom Domains Guide](dns-custom-domains.md)** for complete instructions.

Quick setup:

```bash
# Associate beta subdomain with staging
aws apprunner associate-custom-domain \
    --service-arn ${STAGING_SERVICE_ARN} \
    --domain-name beta.togetherhood.us \
    --region us-east-1

# Get validation records
aws apprunner describe-custom-domains \
    --service-arn ${STAGING_SERVICE_ARN} \
    --region us-east-1

# Add CNAME records in Wix DNS:
# 1. Validation record: _abc.beta → _xyz.acm-validations.aws.
# 2. Beta CNAME: beta → <staging-url>.awsapprunner.com
```

**Benefits of custom domain:**
- Professional URL for stakeholders
- Easier to remember than App Runner URL
- SSL certificate automatically provisioned
- Simulates production environment

### Step 5: Create Staging Deployment GitHub Workflow

**`.github/workflows/cd-staging.yml`:**

```yaml
name: Deploy to Staging

on:
  push:
    branches:
      - develop
  workflow_dispatch:

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: togetherhood-app
  IMAGE_TAG: staging-${{ github.sha }}

jobs:
  build-and-deploy-staging:
    name: Build and Deploy to Staging
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push Docker image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        run: |
          # Build image with staging tag
          docker build \
            -f docker/Dockerfile \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:staging \
            --cache-from type=registry,ref=$ECR_REGISTRY/$ECR_REPOSITORY:staging \
            --cache-to type=inline \
            .

          # Push both tags
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:staging

      - name: Wait for App Runner deployment
        run: |
          echo "Waiting for App Runner staging service to detect and deploy new image..."
          sleep 120

      - name: Verify staging deployment
        env:
          STAGING_URL: ${{ secrets.STAGING_SERVICE_URL }}
        run: |
          # Health check
          echo "Testing staging health endpoint..."
          curl -f https://${STAGING_URL}/api/health || exit 1

          echo "✅ Staging deployment successful!"

      - name: Run smoke tests against staging
        working-directory: apps/e2e
        env:
          E2E_BASE_URL: https://${{ secrets.STAGING_SERVICE_URL }}
          E2E_ADMIN_EMAIL: ${{ secrets.E2E_STAGING_ADMIN_EMAIL }}
          E2E_ADMIN_PASSWORD: ${{ secrets.E2E_STAGING_ADMIN_PASSWORD }}
        run: |
          npm ci
          npm run test:staging -- --spec=smoke-tests.feature || echo "E2E tests failed (non-blocking)"

      - name: Deployment summary
        run: |
          echo "### Staging Deployment Summary 🚀" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "- **Environment:** Staging" >> $GITHUB_STEP_SUMMARY
          echo "- **Branch:** develop" >> $GITHUB_STEP_SUMMARY
          echo "- **Image Tag:** \`${{ env.IMAGE_TAG }}\`" >> $GITHUB_STEP_SUMMARY
          echo "- **Deployed At:** $(date -u +'%Y-%m-%d %H:%M:%S UTC')" >> $GITHUB_STEP_SUMMARY
          echo "- **Staging URL:** https://${{ secrets.STAGING_SERVICE_URL }}" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "Ready for QA testing!" >> $GITHUB_STEP_SUMMARY

      - name: Notify Slack
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
          payload: |
            {
              "text": "${{ job.status == 'success' && '✅ Staging Deployed Successfully' || '❌ Staging Deployment Failed' }}",
              "attachments": [{
                "color": "${{ job.status == 'success' && 'good' || 'danger' }}",
                "fields": [
                  {"title": "Environment", "value": "Staging", "short": true},
                  {"title": "Branch", "value": "develop", "short": true},
                  {"title": "Commit", "value": "${{ github.sha }}", "short": true},
                  {"title": "URL", "value": "https://${{ secrets.STAGING_SERVICE_URL }}", "short": false}
                ]
              }]
            }
```

### Step 6: Configure GitHub Secrets

Add the following secrets to your GitHub repository:

```bash
# Using GitHub CLI
gh secret set STAGING_SERVICE_URL --body "<your-staging-url>"
gh secret set STAGING_SERVICE_ARN --body "<your-staging-arn>"
gh secret set E2E_STAGING_ADMIN_EMAIL --body "admin@example.com"
gh secret set E2E_STAGING_ADMIN_PASSWORD --body "staging_test_password"
gh secret set SLACK_WEBHOOK_URL --body "<your-slack-webhook>" # Optional
```

Or via GitHub UI:
1. Go to **Settings** → **Secrets and variables** → **Actions**
2. Add each secret listed above

### Step 7: Test Staging Deployment

```bash
# 1. Make a change and push to main
git checkout main
echo "# Test staging deployment" >> README.md
git add README.md
git commit -m "test: Trigger deployment"
git push origin main

# 2. Monitor GitHub Actions
gh run list --workflow=cd-production.yml
gh run watch

# 3. Verify staging is deployed (if using manual promotion)
curl https://<staging-url>/api/health

# 4. Test the application
open https://<staging-url>
```

---

## Workflow: Trunk-Based Development

### Typical Development Flow

```
1. Developer creates short-lived feature branch from 'main'
   ↓
2. Implements feature + tests (keep PRs small!)
   ↓
3. Creates PR to 'main'
   ↓
4. PR validation (unit tests, lint, docker build)
   ↓
5. PR approved and merged to 'main'
   ↓
6. 🚀 AUTO-DEPLOY TO PRODUCTION
   ↓
7. Monitor production health
   ↓
8. If issues found: Hotfix PR to 'main' → immediate deploy
```

### Staging Testing Strategy

**Note:** With trunk-based development, `main` is the single source of truth.

**Option A: Automatic Deploy to Both (Recommended)**
- Every merge to `main` deploys to production automatically
- Staging environment is used for pre-merge testing via manual deployments
- Use feature flags to hide incomplete features in production

**Option B: Manual Staging Promotion**
- Deploy specific commits to staging for testing
- After validation, the same commit auto-deploys to production on next merge

**Option C: Tag-based Releases**

```bash
# Create release tag from main
git checkout main
git pull origin main
git tag -a v1.2.0 -m "Release v1.2.0"
git push origin v1.2.0
# Optionally trigger production deployment from tags
```

---

## Managing Staging Data

### Database Seeding

Create seed scripts for staging:

**`apps/api/Scripts/seed-staging.sql`:**

```sql
-- Clear staging data
DELETE FROM sessions;
DELETE FROM courses;
DELETE FROM partners;
DELETE FROM users WHERE email LIKE '%test%';

-- Seed test data
INSERT INTO users (email, password_hash, role) VALUES
('admin@staging.com', 'hashed_password', 'Admin'),
('instructor@staging.com', 'hashed_password', 'Instructor');

INSERT INTO partners (name, email, status) VALUES
('Test Partner 1', 'partner1@staging.com', 'Active'),
('Test Partner 2', 'partner2@staging.com', 'Active');

-- ... more seed data
```

Run manually or via migration:

```bash
# Connect to staging RDS
mysql -h <staging-db-endpoint> -u staging_user -p togetherhood_staging < apps/api/Scripts/seed-staging.sql
```

### Refreshing Staging from Production

To test with production-like data:

```bash
# 1. Create snapshot of production database
aws rds create-db-snapshot \
    --db-instance-identifier togetherhood-production-db \
    --db-snapshot-identifier prod-snapshot-$(date +%Y%m%d) \
    --region us-east-1

# 2. Restore snapshot to staging
aws rds restore-db-instance-from-db-snapshot \
    --db-instance-identifier togetherhood-staging-db-temp \
    --db-snapshot-identifier prod-snapshot-$(date +%Y%m%d) \
    --region us-east-1

# 3. Rename databases (requires downtime)
# ... or use logical dump/restore
```

**Warning:** Sanitize PII/sensitive data after restore!

---

## Monitoring Staging

### CloudWatch Dashboards

Create staging-specific dashboard:

```bash
aws cloudwatch put-dashboard \
    --dashboard-name TogetherhoodAppStaging \
    --dashboard-body file://aws-config/cloudwatch-dashboard-staging.json
```

### CloudWatch Alarms

```bash
# Alarm for staging health check failures
aws cloudwatch put-metric-alarm \
    --alarm-name staging-health-check-failed \
    --alarm-description "Staging health check failing" \
    --metric-name HealthChecksFailed \
    --namespace AWS/AppRunner \
    --statistic Average \
    --period 300 \
    --threshold 1 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --alarm-actions <SNS_TOPIC_ARN>
```

### Logs

```bash
# View staging logs
aws logs tail /aws/apprunner/togetherhood-app-staging/application \
    --follow \
    --region us-east-1

# Filter for errors
aws logs tail /aws/apprunner/togetherhood-app-staging/application \
    --follow \
    --filter-pattern "ERROR" \
    --region us-east-1
```

---

## Cost Optimization

Staging typically costs less than production:

| Resource | Production | Staging | Monthly Cost Difference |
|----------|------------|---------|------------------------|
| **App Runner** | 1 vCPU, 2GB | 0.5 vCPU, 1GB | ~$30 savings |
| **RDS** | db.t3.large | db.t3.micro | ~$100 savings |
| **Data Transfer** | High | Low | ~$20 savings |
| **Backups** | 30 days | 7 days | ~$10 savings |

**Total Staging Cost:** ~$50-80/month (vs ~$200-250 for production)

### Additional Savings

- Stop staging during off-hours (nights/weekends)
- Use Spot instances for staging RDS (if downtime acceptable)
- Reduce backup retention to 1-3 days
- Use smaller S3 bucket with lifecycle policies

---

## Troubleshooting

### Staging deployment fails

**Check:**
1. Verify ECR image with `staging` tag exists
2. Check App Runner service status
3. Review CloudWatch logs for errors
4. Verify secrets are configured correctly
5. Check database connectivity

### Staging shows production data

**Solution:**
- Verify connection string points to staging database
- Check Secrets Manager configuration
- Restart App Runner service to pick up new secrets

### Can't access staging URL

**Solutions:**
- Verify App Runner service is in "RUNNING" state
- Check security groups allow inbound HTTPS
- Verify health check endpoint is responding
- Check DNS if using custom domain

---

## Best Practices

### ✅ DO

- ✅ Deploy to staging before production
- ✅ Run E2E tests against staging
- ✅ Use test/sandbox credentials for external services
- ✅ Keep staging environment close to production
- ✅ Test database migrations in staging first
- ✅ Document differences between staging and production
- ✅ Use staging for demos and QA

### ❌ DON'T

- ❌ Use production credentials in staging
- ❌ Skip testing in staging
- ❌ Deploy directly to production without staging
- ❌ Use staging for load testing (it's smaller)
- ❌ Store real customer data in staging
- ❌ Give everyone production access (staging is safer)

---

## Summary

**Staging Environment Checklist:**

- ✅ Staging RDS database created
- ✅ Staging secrets configured in Secrets Manager
- ✅ Staging IAM roles and policies created
- ✅ Staging App Runner service deployed
- ✅ GitHub workflow for staging deployment
- ✅ GitHub secrets configured
- ✅ Staging deployment tested
- ✅ E2E tests run against staging
- ✅ Monitoring and alerts configured
- ✅ Team trained on develop → staging → production flow

**Next Steps:**
1. Test staging deployment end-to-end
2. Run E2E test suite against staging
3. Validate database migrations in staging
4. Promote to production after validation

**Related Guides:**
- [AWS Infrastructure Setup](aws-infrastructure-setup.md)
- [AWS Secrets Manager Integration](aws-secrets-manager.md)
- [CI/CD Testing Strategy](cicd-testing.md)
- [E2E Testing Integration](e2e-testing.md)
