# AWS Secrets Manager Integration Guide

## Overview

This guide covers integrating AWS Secrets Manager with the Togetherhood monorepo for secure secrets management in production.

### Why Secrets Manager?

✅ **No secrets in Docker images** - Secrets never baked into containers
✅ **No secrets in GitHub** - Keeps secrets out of source control
✅ **Runtime injection** - App Runner injects secrets at container startup
✅ **Automatic rotation** - Built-in support for credential rotation
✅ **Audit logging** - CloudWatch tracks all secret access
✅ **Encryption at rest** - KMS encryption by default

---

## Architecture

```
GitHub Actions
    ↓ (Build Docker image - NO SECRETS)
AWS ECR
    ↓ (Pull image)
AWS App Runner
    ↓ (Fetch secrets from Secrets Manager)
.NET Application
    ↓ (Read from environment variables)
Configuration
```

---

## Setup Steps

### Step 1: Create Secrets in AWS Secrets Manager

#### Option A: Using AWS Console

1. Navigate to **AWS Secrets Manager** → **Store a new secret**
2. Choose **Other type of secret**
3. Use **Key/value** format
4. Add your secrets:

```json
{
  "ConnectionStrings__DefaultConnection": "Server=your-rds-host.us-east-1.rds.amazonaws.com;Database=togetherhood;User=app_user;Password=<strong-password>",
  "JwtSettings__Secret": "<generate-with: openssl rand -base64 64>",
  "JwtSettings__Issuer": "TogetherhoodAPI",
  "JwtSettings__Audience": "TogetherhoodWeb",
  "SendGrid__ApiKey": "SG.xxxxxxxxxxxx",
  "SendGrid__FromEmail": "noreply@togetherhood.com",
  "Twilio__AccountSid": "ACxxxxxxxxxxxx",
  "Twilio__AuthToken": "your-auth-token",
  "AWS__AccessKeyId": "AKIAxxxxxxxxxxxx",
  "AWS__SecretAccessKey": "xxxxxxxxxxxx",
  "Tipalti__ApiKey": "your-tipalti-key"
}
```

5. Name the secret: `togetherhood-app/production/config`
6. Enable automatic rotation if applicable
7. Store the secret

#### Option B: Using AWS CLI

Create a file `aws-config/secrets.json`:

```json
{
  "ConnectionStrings__DefaultConnection": "Server=...",
  "JwtSettings__Secret": "...",
  "SendGrid__ApiKey": "...",
  "Twilio__AccountSid": "...",
  "Twilio__AuthToken": "..."
}
```

Create the secret:

```bash
aws secretsmanager create-secret \
    --name togetherhood-app/production/config \
    --description "Production configuration for Togetherhood App" \
    --secret-string file://aws-config/secrets.json \
    --region us-east-1
```

**Important:** Add `aws-config/secrets.json` to `.gitignore`!

### Step 2: Create IAM Role for App Runner

Create IAM policy that allows App Runner to read secrets:

**File: `aws-config/app-runner-secrets-policy.json`**

```json
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
        "arn:aws:secretsmanager:us-east-1:<ACCOUNT_ID>:secret:togetherhood-app/production/config*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": [
        "arn:aws:kms:us-east-1:<ACCOUNT_ID>:key/*"
      ],
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "secretsmanager.us-east-1.amazonaws.com"
        }
      }
    }
  ]
}
```

Replace `<ACCOUNT_ID>` with your AWS account ID:

```bash
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
sed -i "s/<ACCOUNT_ID>/${AWS_ACCOUNT_ID}/g" aws-config/app-runner-secrets-policy.json
```

Create and attach the policy:

```bash
# Create policy
aws iam create-policy \
    --policy-name TogetherhoodAppRunnerSecretsPolicy \
    --policy-document file://aws-config/app-runner-secrets-policy.json

# Get the policy ARN
POLICY_ARN=$(aws iam list-policies \
    --query "Policies[?PolicyName=='TogetherhoodAppRunnerSecretsPolicy'].Arn" \
    --output text)

# Create instance role for App Runner
aws iam create-role \
    --role-name TogetherhoodAppRunnerInstanceRole \
    --assume-role-policy-document '{
      "Version": "2012-10-17",
      "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "tasks.apprunner.amazonaws.com"},
        "Action": "sts:AssumeRole"
      }]
    }'

# Attach policy to role
aws iam attach-role-policy \
    --role-name TogetherhoodAppRunnerInstanceRole \
    --policy-arn ${POLICY_ARN}

# Get role ARN (save this for App Runner configuration)
INSTANCE_ROLE_ARN=$(aws iam get-role \
    --role-name TogetherhoodAppRunnerInstanceRole \
    --query 'Role.Arn' \
    --output text)

echo "Instance Role ARN: ${INSTANCE_ROLE_ARN}"
```

### Step 3: Configure App Runner Service with Secrets

When creating or updating the App Runner service, reference secrets:

```bash
aws apprunner update-service \
    --service-arn <YOUR_SERVICE_ARN> \
    --instance-configuration InstanceRoleArn=${INSTANCE_ROLE_ARN} \
    --source-configuration '{
      "ImageRepository": {
        "ImageConfiguration": {
          "Port": "8080",
          "RuntimeEnvironmentVariables": {
            "ASPNETCORE_ENVIRONMENT": "Production"
          },
          "RuntimeEnvironmentSecrets": {
            "ConnectionStrings__DefaultConnection": "arn:aws:secretsmanager:us-east-1:<ACCOUNT_ID>:secret:togetherhood-app/production/config:ConnectionStrings__DefaultConnection::",
            "JwtSettings__Secret": "arn:aws:secretsmanager:us-east-1:<ACCOUNT_ID>:secret:togetherhood-app/production/config:JwtSettings__Secret::",
            "SendGrid__ApiKey": "arn:aws:secretsmanager:us-east-1:<ACCOUNT_ID>:secret:togetherhood-app/production/config:SendGrid__ApiKey::",
            "Twilio__AccountSid": "arn:aws:secretsmanager:us-east-1:<ACCOUNT_ID>:secret:togetherhood-app/production/config:Twilio__AccountSid::",
            "Twilio__AuthToken": "arn:aws:secretsmanager:us-east-1:<ACCOUNT_ID>:secret:togetherhood-app/production/config:Twilio__AuthToken::"
          }
        }
      }
    }' \
    --region us-east-1
```

**Format Explanation:**

The secret ARN format for runtime secrets is:
```
arn:aws:secretsmanager:<region>:<account-id>:secret:<secret-name>:<json-key>::
```

The trailing `::` is required and means "use the latest version."

### Step 4: Verify .NET Application Reads Configuration

Your .NET application should already read from environment variables via the standard configuration system. Verify `Program.cs` includes:

```csharp
var builder = WebApplication.CreateBuilder(args);

// This automatically reads from:
// 1. appsettings.json
// 2. appsettings.{Environment}.json
// 3. Environment variables (highest priority)
// 4. User secrets (development only)

// Access configuration
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
var jwtSecret = builder.Configuration["JwtSettings:Secret"];
```

**Important:** Environment variable naming convention:
- JSON path: `ConnectionStrings:DefaultConnection`
- Environment variable: `ConnectionStrings__DefaultConnection` (double underscore)

---

## Secret Organization Strategies

### Strategy 1: Single Secret with All Configuration (Recommended)

**Secret Name:** `togetherhood-app/production/config`

**Contents:**
```json
{
  "ConnectionStrings__DefaultConnection": "...",
  "JwtSettings__Secret": "...",
  "JwtSettings__Issuer": "...",
  "SendGrid__ApiKey": "...",
  "Twilio__AccountSid": "...",
  "Twilio__AuthToken": "...",
  "AWS__AccessKeyId": "...",
  "AWS__SecretAccessKey": "..."
}
```

**Pros:**
- ✅ Single secret to manage
- ✅ Simpler IAM policy
- ✅ Easier to reference in App Runner

**Cons:**
- ❌ All secrets rotate together
- ❌ Harder to give granular access

### Strategy 2: Separate Secrets by Service

**Secrets:**
- `togetherhood-app/production/database`
- `togetherhood-app/production/jwt`
- `togetherhood-app/production/sendgrid`
- `togetherhood-app/production/twilio`
- `togetherhood-app/production/aws`

**Pros:**
- ✅ Granular access control
- ✅ Independent rotation
- ✅ Easier to share specific secrets

**Cons:**
- ❌ More secrets to manage
- ❌ More complex IAM policies
- ❌ More App Runner configuration

**Recommendation:** Use Strategy 1 for simplicity, switch to Strategy 2 if you need granular access control.

---

## Updating Secrets

### Using AWS Console

1. Navigate to **Secrets Manager** → Find your secret
2. Click **Retrieve secret value**
3. Click **Edit**
4. Update values
5. Click **Save**
6. **Restart App Runner service** to pick up changes:

```bash
aws apprunner update-service \
    --service-arn <YOUR_SERVICE_ARN> \
    --region us-east-1
```

### Using AWS CLI

```bash
# Update entire secret
aws secretsmanager update-secret \
    --secret-id togetherhood-app/production/config \
    --secret-string file://aws-config/secrets-updated.json \
    --region us-east-1

# Update single value (requires jq)
aws secretsmanager get-secret-value \
    --secret-id togetherhood-app/production/config \
    --region us-east-1 \
    --query SecretString \
    --output text | \
jq '.["SendGrid__ApiKey"] = "new-api-key"' | \
aws secretsmanager update-secret \
    --secret-id togetherhood-app/production/config \
    --secret-string file:///dev/stdin \
    --region us-east-1

# Restart App Runner to pick up changes
aws apprunner update-service \
    --service-arn <YOUR_SERVICE_ARN> \
    --region us-east-1
```

---

## Secret Rotation

### Automatic Rotation for RDS Passwords

For database passwords, enable automatic rotation:

```bash
aws secretsmanager rotate-secret \
    --secret-id togetherhood-app/production/database \
    --rotation-lambda-arn arn:aws:lambda:us-east-1:<ACCOUNT_ID>:function:SecretsManagerRDSRotation \
    --rotation-rules AutomaticallyAfterDays=30 \
    --region us-east-1
```

This requires:
1. Lambda function for rotation (AWS provides templates)
2. Lambda has access to RDS and Secrets Manager
3. Database user has permissions to change passwords

See: [AWS RDS Password Rotation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotate-secrets_turn-on-for-db.html)

### Manual Rotation for API Keys

For external service API keys (SendGrid, Twilio, etc.):

1. Generate new API key in external service
2. Update secret in Secrets Manager
3. Restart App Runner service
4. Verify application works with new key
5. Revoke old API key in external service

---

## Monitoring & Auditing

### CloudWatch Logs

All secret access is logged to CloudWatch:

```bash
# View secret access logs
aws logs tail /aws/secretsmanager/togetherhood-app/production/config \
    --follow \
    --region us-east-1
```

### CloudTrail Events

Track all Secrets Manager API calls:

```bash
# View recent Secrets Manager events
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=ResourceType,AttributeValue=AWS::SecretsManager::Secret \
    --max-results 50 \
    --region us-east-1
```

### Setup Alarms

Create CloudWatch alarm for unauthorized access attempts:

```bash
aws cloudwatch put-metric-alarm \
    --alarm-name togetherhood-secrets-unauthorized-access \
    --alarm-description "Alert on unauthorized Secrets Manager access" \
    --metric-name UnauthorizedAccess \
    --namespace AWS/SecretsManager \
    --statistic Sum \
    --period 300 \
    --threshold 1 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 1 \
    --region us-east-1
```

---

## Local Development vs Production

### Local Development

Use `.env` files (NOT in git):

**`apps/api/.env.local`:**
```env
ConnectionStrings__DefaultConnection=Server=localhost;Database=togetherhood_api;User=togetherhood;Password=local_dev_pass
JwtSettings__Secret=local-dev-secret-min-32-chars-long
SendGrid__ApiKey=SG.test-key
```

Add to `.gitignore`:
```gitignore
**/.env.local
**/.env.*.local
**/appsettings.Local.json
```

### Production

Secrets come from AWS Secrets Manager (via App Runner).

### Staging Environment

Create staging secrets with **test/sandbox** credentials:

```bash
# Create staging secret
aws secretsmanager create-secret \
    --name togetherhood-app/staging/config \
    --description "Staging configuration for Togetherhood App" \
    --secret-string '{
      "ConnectionStrings__DefaultConnection": "Server=staging-db.us-east-1.rds.amazonaws.com;Database=togetherhood_staging;User=staging_user;Password=<staging-password>",
      "JwtSettings__Secret": "<different-from-prod-secret>",
      "SendGrid__ApiKey": "<SENDGRID_SANDBOX_KEY>",
      "Twilio__AccountSid": "<TWILIO_TEST_SID>",
      "Twilio__AuthToken": "<TWILIO_TEST_TOKEN>",
      "Tipalti__ApiUrl": "https://ui2.sandbox.tipalti.com/"
    }' \
    --region us-east-1
```

**Important:** Use sandbox/test API keys in staging!

### Development AWS Environments (Optional)

If you need a third environment for development:

```bash
aws secretsmanager create-secret \
    --name togetherhood-app/development/config \
    --description "Development environment configuration" \
    --secret-string file://aws-config/secrets-dev.json \
    --region us-east-1
```

### Secret Naming Convention

| Environment | Secret Name |
|-------------|------------|
| **Local** | `.env.local` files (not AWS) |
| **Development** | `togetherhood-app/development/config` |
| **Staging** | `togetherhood-app/staging/config` |
| **Production** | `togetherhood-app/production/config` |

Each App Runner environment references its own secret via the `RuntimeEnvironmentSecrets` configuration.

---

## Security Best Practices

### ✅ DO

- ✅ Use different secrets for each environment (dev/staging/prod)
- ✅ Rotate secrets regularly (especially database passwords)
- ✅ Use strong, randomly generated secrets (min 32 characters)
- ✅ Enable automatic rotation where possible
- ✅ Monitor CloudTrail for unauthorized access
- ✅ Use least-privilege IAM policies
- ✅ Encrypt secrets at rest (default with Secrets Manager)
- ✅ Audit secret access logs regularly

### ❌ DON'T

- ❌ Hardcode secrets in application code
- ❌ Commit secrets to git (even private repos)
- ❌ Bake secrets into Docker images
- ❌ Share secrets via email or chat
- ❌ Use the same secrets across environments
- ❌ Grant overly broad IAM permissions
- ❌ Store secrets in environment variables in GitHub Actions
- ❌ Use weak or default passwords

---

## Troubleshooting

### Issue: App Runner can't read secrets

**Error:**
```
Unable to retrieve secret value for arn:aws:secretsmanager:...
```

**Solutions:**

1. Verify IAM instance role is attached to App Runner service:
```bash
aws apprunner describe-service \
    --service-arn <YOUR_SERVICE_ARN> \
    --region us-east-1 \
    --query 'Service.InstanceConfiguration.InstanceRoleArn'
```

2. Verify role has permission to read secrets:
```bash
aws iam list-attached-role-policies \
    --role-name TogetherhoodAppRunnerInstanceRole
```

3. Verify secret exists and ARN is correct:
```bash
aws secretsmanager describe-secret \
    --secret-id togetherhood-app/production/config \
    --region us-east-1
```

4. Check CloudWatch logs for detailed error:
```bash
aws logs tail /aws/apprunner/<service-name>/application \
    --follow \
    --region us-east-1
```

### Issue: Application can't find configuration values

**Error:**
```
Configuration key 'JwtSettings:Secret' not found
```

**Solutions:**

1. Verify environment variable name matches configuration path:
   - Configuration: `JwtSettings:Secret`
   - Environment variable: `JwtSettings__Secret` (double underscore!)

2. Check App Runner service environment configuration:
```bash
aws apprunner describe-service \
    --service-arn <YOUR_SERVICE_ARN> \
    --region us-east-1 \
    --query 'Service.SourceConfiguration.ImageRepository.ImageConfiguration'
```

3. Verify secret contains the expected key:
```bash
aws secretsmanager get-secret-value \
    --secret-id togetherhood-app/production/config \
    --region us-east-1 \
    --query SecretString \
    --output text | jq
```

### Issue: Secrets Manager cost too high

**Symptoms:** AWS bill shows high Secrets Manager charges

**Solutions:**

1. Consolidate secrets - Use single secret instead of many
2. Reduce API calls - Cache configuration in application
3. Use Parameter Store for non-sensitive config - Cheaper alternative
4. Delete unused secrets:
```bash
aws secretsmanager list-secrets --region us-east-1 | jq -r '.SecretList[].Name'
aws secretsmanager delete-secret --secret-id <unused-secret> --region us-east-1
```

**Pricing Reference:**
- $0.40 per secret per month
- $0.05 per 10,000 API calls
- Free tier: 30-day trial for new secrets

---

## Migration from Hardcoded Secrets

### Current State (Example)

**appsettings.Production.json:**
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=prod-db;Database=togetherhood;User=app;Password=HARDCODED_PASSWORD"
  },
  "JwtSettings": {
    "Secret": "HARDCODED_JWT_SECRET"
  }
}
```

### Target State

**appsettings.Production.json:**
```json
{
  "ConnectionStrings": {
    "DefaultConnection": ""
  },
  "JwtSettings": {
    "Secret": ""
  }
}
```

**Secrets Manager:**
```json
{
  "ConnectionStrings__DefaultConnection": "Server=prod-db;Database=togetherhood;User=app;Password=STRONG_PASSWORD",
  "JwtSettings__Secret": "STRONG_RANDOM_SECRET"
}
```

### Migration Steps

1. **Extract all secrets** from appsettings.Production.json
2. **Create Secrets Manager secret** with all values
3. **Update appsettings.Production.json** to remove secrets
4. **Configure App Runner** to inject secrets
5. **Test in staging** environment first
6. **Deploy to production**
7. **Verify application works**
8. **Remove old secrets** from any other locations

---

## Reference

### Environment Variable Naming

| Configuration Path | Environment Variable |
|--------------------|---------------------|
| `ConnectionStrings:DefaultConnection` | `ConnectionStrings__DefaultConnection` |
| `JwtSettings:Secret` | `JwtSettings__Secret` |
| `JwtSettings:Issuer` | `JwtSettings__Issuer` |
| `SendGrid:ApiKey` | `SendGrid__ApiKey` |
| `AWS:Region` | `AWS__Region` |
| `AWS:S3BucketName` | `AWS__S3BucketName` |

Rule: Replace `:` with `__` (double underscore)

### Useful Commands

```bash
# Create secret
aws secretsmanager create-secret --name <name> --secret-string file://secrets.json

# Get secret value
aws secretsmanager get-secret-value --secret-id <name> --query SecretString --output text | jq

# Update secret
aws secretsmanager update-secret --secret-id <name> --secret-string file://secrets.json

# List all secrets
aws secretsmanager list-secrets

# Delete secret (with recovery window)
aws secretsmanager delete-secret --secret-id <name> --recovery-window-in-days 7

# Restore deleted secret
aws secretsmanager restore-secret --secret-id <name>

# Describe secret
aws secretsmanager describe-secret --secret-id <name>
```

---

## Next Steps

- ✅ Set up secrets in AWS Secrets Manager
- ✅ Configure IAM roles and policies
- ✅ Update App Runner service configuration
- ✅ Test secret retrieval in staging
- ✅ Monitor CloudWatch logs for issues
- ✅ Document secret rotation procedures
- ✅ Train team on secret management

**Related Guides:**
- [AWS Infrastructure Setup](aws-infrastructure-setup.md)
- [CI/CD Testing Strategy](cicd-testing.md)
- [Migration Checklist](migration-checklist.md) - See Environment Variables section
