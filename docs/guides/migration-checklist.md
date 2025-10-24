# Migration Considerations Checklist

## Overview

This document covers additional considerations beyond the core migration that you should address for a successful production deployment.

---

## 🗄️ Database Considerations

### Database Migrations

**Question:** How will database migrations run in staging/production?

**Current State:**
- Using Flyway for migrations (mentioned in API docs)
- Entity Framework migrations available

**Considerations:**

#### Option 1: Run Migrations in Dockerfile (Not Recommended)
```dockerfile
# In Dockerfile
RUN dotnet ef database update
```
**Problem:** Fails if DB not accessible during build

#### Option 2: Run Migrations at Container Startup (Risky)
```csharp
// Program.cs
using (var scope = app.Services.CreateScope())
{
    var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
    context.Database.Migrate();
}
```
**Problem:** Race condition if multiple containers start simultaneously

#### Option 3: Separate Migration Job (Recommended)
```yaml
# .github/workflows/cd-production.yml
- name: Run database migrations
  run: |
    # Connect to RDS from GitHub Actions
    docker run \
      -e ConnectionStrings__DefaultConnection="${DB_CONNECTION}" \
      togetherhood-app \
      dotnet ef database update --project Togetherhood.Data
```

#### Option 4: Manual Migrations with Approval
```yaml
# Require manual approval before migration
- name: Wait for migration approval
  uses: trstringer/manual-approval@v1

- name: Run migrations
  run: |
    # Run migration script
```

**Recommendation:** Use Option 3 or 4 for production. Add to workflow:

```yaml
jobs:
  migrate-database:
    name: Run Database Migrations
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '6.0.x'

      - name: Install dotnet-ef
        run: dotnet tool install --global dotnet-ef

      - name: Run migrations
        working-directory: apps/api
        env:
          ConnectionStrings__DefaultConnection: ${{ secrets.PROD_DB_CONNECTION }}
        run: dotnet ef database update --project Togetherhood.Data --startup-project Togetherhood.API

  deploy:
    needs: migrate-database
    # ... rest of deployment
```

### Data Migration from Old Infrastructure

**Question:** How to migrate existing data to new RDS instances?

**Steps:**
1. **Export from old database:**
   ```bash
   mysqldump -h old-db-host -u user -p togetherhood > backup.sql
   ```

2. **Import to staging:**
   ```bash
   mysql -h staging-db.rds.amazonaws.com -u admin -p togetherhood_staging < backup.sql
   ```

3. **Test application against staging data**

4. **Schedule production data migration:**
   - Announce maintenance window
   - Put old app in read-only mode
   - Export latest data
   - Import to production RDS
   - Verify data integrity
   - Switch DNS to new infrastructure
   - Monitor closely

**Considerations:**
- Data size (how long will import take?)
- Downtime acceptable?
- Data sanitization needed for staging?
- Test rollback procedure

### Connection Pooling

**Current:** Entity Framework default pooling

**Recommendation:** Configure explicit pooling for App Runner:

```csharp
// Program.cs or Startup.cs
services.AddDbContext<ApplicationDbContext>(options =>
{
    options.UseMySql(
        connectionString,
        new MySqlServerVersion(new Version(8, 0, 35)),
        mySqlOptions =>
        {
            mySqlOptions.EnableRetryOnFailure(
                maxRetryCount: 3,
                maxRetryDelay: TimeSpan.FromSeconds(5),
                errorNumbersToAdd: null);

            // Connection pooling
            mySqlOptions.CommandTimeout(30);
        }
    );
});
```

**Connection string with pooling:**
```
Server=db-host;Database=togetherhood;User=user;Password=pass;Pooling=true;MinPoolSize=5;MaxPoolSize=100;ConnectionTimeout=30;
```

---

## 📦 S3 Buckets for File Uploads

**Mentioned in IAM policies but not created:**

```bash
# Create staging S3 bucket
aws s3 mb s3://togetherhood-staging-uploads --region us-east-1

aws s3api put-bucket-encryption \
    --bucket togetherhood-staging-uploads \
    --server-side-encryption-configuration '{
      "Rules": [{
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "AES256"
        }
      }]
    }'

aws s3api put-public-access-block \
    --bucket togetherhood-staging-uploads \
    --public-access-block-configuration \
        "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Create production S3 bucket
aws s3 mb s3://togetherhood-production-uploads --region us-east-1
# ... same encryption and access block settings

# Add lifecycle policy for staging (optional - auto-delete old files)
aws s3api put-bucket-lifecycle-configuration \
    --bucket togetherhood-staging-uploads \
    --lifecycle-configuration '{
      "Rules": [{
        "Id": "DeleteOldFiles",
        "Status": "Enabled",
        "ExpirationInDays": 90,
        "Filter": {"Prefix": ""}
      }]
    }'
```

**Add to Secrets Manager:**
```json
{
  "AWS__S3BucketName": "togetherhood-production-uploads",
  "AWS__S3Region": "us-east-1"
}
```

---

## 📊 Comprehensive Monitoring & Observability

### Application Performance Monitoring (APM)

**Options:**

1. **AWS X-Ray** (Native AWS)
   ```bash
   # Add to IAM instance role
   {
     "Effect": "Allow",
     "Action": [
       "xray:PutTraceSegments",
       "xray:PutTelemetryRecords"
     ],
     "Resource": "*"
   }
   ```

2. **Datadog** (Third-party)
   - Add Datadog agent sidecar to App Runner
   - Comprehensive APM, logs, metrics

3. **New Relic** (Third-party)
   - .NET agent integration
   - Full stack monitoring

**Recommendation:** Start with CloudWatch + X-Ray (free tier), add Datadog later if needed.

### Structured Logging

**Current:** Likely using Console.WriteLine or basic logging

**Recommendation:** Use Serilog with structured logging:

```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.CloudWatch
```

```csharp
// Program.cs
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .Enrich.FromLogContext()
    .Enrich.WithProperty("Environment", Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT"))
    .WriteTo.Console(new JsonFormatter())
    .WriteTo.AmazonCloudWatch(
        logGroup: "/aws/apprunner/togetherhood-app-production",
        region: RegionEndpoint.USEast1)
    .CreateLogger();

builder.Host.UseSerilog();
```

**Benefits:**
- JSON structured logs (easier to search)
- Correlation IDs for request tracing
- Automatic context enrichment
- Better CloudWatch Insights queries

### CloudWatch Dashboards

Create comprehensive dashboards:

```bash
# CPU/Memory
# Request rate, latency (p50, p95, p99)
# Error rates (4xx, 5xx)
# Database query time
# Active connections
# Custom business metrics
```

### Custom Metrics

**Recommendation:** Send custom metrics to CloudWatch:

```csharp
// Example: Track partner creation
using Amazon.CloudWatch;
using Amazon.CloudWatch.Model;

public async Task CreatePartner(Partner partner)
{
    var cloudWatch = new AmazonCloudWatchClient();

    await cloudWatch.PutMetricDataAsync(new PutMetricDataRequest
    {
        Namespace = "Togetherhood/Business",
        MetricData = new List<MetricDatum>
        {
            new MetricDatum
            {
                MetricName = "PartnerCreated",
                Value = 1,
                Unit = StandardUnit.Count,
                TimestampUtc = DateTime.UtcNow
            }
        }
    });

    // ... rest of creation logic
}
```

---

## 🔒 Security Considerations

### VPC Configuration

**Current:** App Runner and RDS likely in default VPC

**Recommendation:** Use private VPC for production:

```bash
# Create VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=togetherhood-vpc}]'

# Create private subnets
aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.1.0/24 --availability-zone us-east-1a
aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.2.0/24 --availability-zone us-east-1b

# Create App Runner VPC connector
aws apprunner create-vpc-connector \
    --vpc-connector-name togetherhood-vpc-connector \
    --subnets <SUBNET_ID_1> <SUBNET_ID_2> \
    --security-groups <SECURITY_GROUP_ID>

# Update App Runner service to use VPC connector
aws apprunner update-service \
    --service-arn <SERVICE_ARN> \
    --network-configuration EgressConfiguration={EgressType=VPC,VpcConnectorArn=<VPC_CONNECTOR_ARN>}
```

### WAF (Web Application Firewall)

**Consideration:** App Runner doesn't support WAF directly

**Options:**
1. Add CloudFront in front of App Runner → Attach WAF to CloudFront
2. Use API Gateway → App Runner → Add WAF to API Gateway
3. Accept risk for Phase 1, add later

### Rate Limiting

**In-app rate limiting:**

```csharp
// Add AspNetCoreRateLimit
dotnet add package AspNetCoreRateLimit

// Startup.cs
services.AddMemoryCache();
services.Configure<IpRateLimitOptions>(options =>
{
    options.GeneralRules = new List<RateLimitRule>
    {
        new RateLimitRule
        {
            Endpoint = "*",
            Period = "1m",
            Limit = 60
        }
    };
});
services.AddSingleton<IRateLimitCounterStore, MemoryCacheRateLimitCounterStore>();
services.AddSingleton<IIpPolicyStore, MemoryCacheIpPolicyStore>();
services.AddSingleton<IRateLimitConfiguration, RateLimitConfiguration>();

// Program.cs
app.UseIpRateLimiting();
```

### Security Headers

```csharp
// Program.cs
app.Use(async (context, next) =>
{
    context.Response.Headers.Add("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Add("X-Frame-Options", "DENY");
    context.Response.Headers.Add("X-XSS-Protection", "1; mode=block");
    context.Response.Headers.Add("Referrer-Policy", "strict-origin-when-cross-origin");
    context.Response.Headers.Add("Permissions-Policy", "geolocation=(), microphone=(), camera=()");
    context.Response.Headers.Add("Content-Security-Policy", "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';");
    context.Response.Headers.Add("Strict-Transport-Security", "max-age=31536000; includeSubDomains");

    await next();
});
```

---

## 💰 Cost Management

### AWS Budgets

```bash
# Create budget alert
aws budgets create-budget \
    --account-id ${AWS_ACCOUNT_ID} \
    --budget '{
      "BudgetName": "togetherhood-monthly-budget",
      "BudgetLimit": {
        "Amount": "500",
        "Unit": "USD"
      },
      "TimeUnit": "MONTHLY",
      "BudgetType": "COST"
    }' \
    --notifications-with-subscribers '[
      {
        "Notification": {
          "NotificationType": "ACTUAL",
          "ComparisonOperator": "GREATER_THAN",
          "Threshold": 80,
          "ThresholdType": "PERCENTAGE"
        },
        "Subscribers": [{
          "SubscriptionType": "EMAIL",
          "Address": "team@togetherhood.com"
        }]
      }
    ]'
```

### Cost Optimization

**Staging:**
- Use smaller RDS instance (db.t3.micro)
- Stop RDS during nights/weekends (manual or Lambda)
- Reduce backup retention to 3 days
- Use lifecycle policies on S3

**Production:**
- Use Reserved Instances for RDS (1-year commitment = 40% savings)
- Use Savings Plans for App Runner
- Monitor unused resources (old ECR images, snapshots)
- Set up cost anomaly detection

---

## 🔄 Caching Strategy

**Question:** Is caching needed?

**Options:**

1. **In-Memory Caching (Simple)**
   ```csharp
   services.AddMemoryCache();
   ```
   **Limitation:** Each App Runner instance has separate cache

2. **Redis/ElastiCache (Recommended for production)**
   ```bash
   # Create ElastiCache Redis cluster
   aws elasticache create-cache-cluster \
       --cache-cluster-id togetherhood-redis \
       --engine redis \
       --cache-node-type cache.t3.micro \
       --num-cache-nodes 1 \
       --security-group-ids <SG_ID> \
       --subnet-group-name <SUBNET_GROUP>
   ```

   ```csharp
   // Add StackExchange.Redis
   services.AddStackExchangeRedisCache(options =>
   {
       options.Configuration = Configuration["Redis:ConnectionString"];
       options.InstanceName = "Togetherhood_";
   });
   ```

3. **CloudFront (for static assets)**
   - Add CloudFront in front of App Runner
   - Cache static assets at edge
   - Reduce load on App Runner

**Recommendation:** Start without Redis, add if performance issues arise.

---

## 🔔 Alerting & Notifications

### Who Gets Alerted?

**Create SNS topics:**

```bash
# Critical alerts (PagerDuty or on-call team)
aws sns create-topic --name togetherhood-critical-alerts

# Warning alerts (team Slack channel)
aws sns create-topic --name togetherhood-warning-alerts

# Info alerts (monitoring dashboard)
aws sns create-topic --name togetherhood-info-alerts
```

### Slack Integration

```bash
# Create Slack webhook in Slack settings
# Subscribe SNS to Slack webhook using AWS Chatbot

aws chatbot create-slack-channel-configuration \
    --configuration-name togetherhood-alerts \
    --iam-role-arn <CHATBOT_ROLE_ARN> \
    --slack-channel-id <CHANNEL_ID> \
    --slack-workspace-id <WORKSPACE_ID> \
    --sns-topic-arns <TOPIC_ARN>
```

### Alert Levels

**Critical (PagerDuty):**
- All instances down
- Database unreachable
- 5xx error rate > 5%
- Health check failing for 5 minutes

**Warning (Slack):**
- High error rate (>2%)
- High latency (p95 > 2s)
- High CPU/Memory (>80%)
- Disk space low

**Info (Dashboard):**
- Deployments
- Scaling events
- Certificate renewals

---

## 🌐 CORS Configuration

**Complete CORS setup:**

```csharp
// Program.cs
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        if (builder.Environment.IsDevelopment())
        {
            // Local development
            policy.WithOrigins(
                "http://localhost:3000",
                "http://localhost:5000"
            );
        }
        else if (builder.Environment.IsStaging())
        {
            // Staging
            policy.WithOrigins(
                "https://beta.togetherhood.us",
                "https://*.awsapprunner.com"  // App Runner staging URLs
            );
        }
        else
        {
            // Production
            policy.WithOrigins(
                "https://app.togetherhood.us",
                "https://togetherhood.us",
                "https://beta.togetherhood.us"
            );
        }

        policy.AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials()
              .WithExposedHeaders("Content-Disposition"); // For file downloads
    });
});

app.UseCors();
```

---

## 🔧 Health Check Endpoint

**Current:** Basic `/api/health` endpoint

**Recommendation:** Comprehensive health check:

```csharp
// Controllers/HealthController.cs
[ApiController]
[Route("api/[controller]")]
public class HealthController : ControllerBase
{
    private readonly ApplicationDbContext _context;
    private readonly IConfiguration _configuration;

    [HttpGet]
    public async Task<IActionResult> Get()
    {
        var health = new
        {
            status = "healthy",
            timestamp = DateTime.UtcNow,
            version = Assembly.GetExecutingAssembly().GetName().Version?.ToString(),
            environment = _configuration["ASPNETCORE_ENVIRONMENT"]
        };

        return Ok(health);
    }

    [HttpGet("detailed")]
    public async Task<IActionResult> GetDetailed()
    {
        var checks = new Dictionary<string, object>();

        // Database connectivity
        try
        {
            var canConnect = await _context.Database.CanConnectAsync();
            checks["database"] = new { status = canConnect ? "healthy" : "unhealthy" };
        }
        catch (Exception ex)
        {
            checks["database"] = new { status = "unhealthy", error = ex.Message };
        }

        // S3 connectivity (if applicable)
        try
        {
            // Test S3 connection
            checks["s3"] = new { status = "healthy" };
        }
        catch (Exception ex)
        {
            checks["s3"] = new { status = "unhealthy", error = ex.Message };
        }

        // External APIs (SendGrid, Twilio, etc.)
        // ... add checks for critical dependencies

        var overallHealthy = checks.Values.All(c =>
            ((dynamic)c).status == "healthy");

        return overallHealthy ? Ok(checks) : StatusCode(503, checks);
    }

    [HttpGet("ready")]
    public IActionResult GetReadiness()
    {
        // Kubernetes-style readiness probe
        // Returns 200 if app is ready to receive traffic
        return Ok(new { status = "ready" });
    }

    [HttpGet("live")]
    public IActionResult GetLiveness()
    {
        // Kubernetes-style liveness probe
        // Returns 200 if app is alive (even if not ready)
        return Ok(new { status = "alive" });
    }
}
```

**Update App Runner health check:**
```bash
aws apprunner update-service \
    --service-arn <SERVICE_ARN> \
    --health-check-configuration '{
      "Protocol": "HTTP",
      "Path": "/api/health",
      "Interval": 10,
      "Timeout": 5,
      "HealthyThreshold": 1,
      "UnhealthyThreshold": 3
    }'
```

---

## 🚀 Zero-Downtime Deployments

**App Runner handles this automatically, but verify:**

1. **Rolling deployments:** App Runner starts new instances before stopping old ones
2. **Health check grace period:** Waits for health check to pass
3. **Traffic cutover:** Gradual traffic shift to new instances

**Test this:**
```bash
# During deployment, continuously hit endpoint
while true; do
  curl -s https://beta.togetherhood.us/api/health | jq .timestamp
  sleep 1
done

# Should see no errors during deployment
```

---

## 📝 Environment Variables - Complete List

**Create comprehensive .env.example:**

```bash
# apps/api/.env.example
# ======================
# Server Configuration
# ======================
ASPNETCORE_ENVIRONMENT=Development
ASPNETCORE_URLS=http://+:5000

# ======================
# Database
# ======================
ConnectionStrings__DefaultConnection=Server=localhost;Database=togetherhood;User=root;Password=password;

# ======================
# JWT Authentication
# ======================
JwtSettings__Secret=your-secret-key-min-32-characters
JwtSettings__Issuer=TogetherhoodAPI
JwtSettings__Audience=TogetherhoodWeb
JwtSettings__ExpirationMinutes=60

# ======================
# AWS Services
# ======================
AWS__Region=us-east-1
AWS__AccessKeyId=your-access-key
AWS__SecretAccessKey=your-secret-key
AWS__S3BucketName=togetherhood-uploads

# ======================
# SendGrid (Email)
# ======================
SendGrid__ApiKey=SG.your-api-key
SendGrid__FromEmail=noreply@togetherhood.com
SendGrid__FromName=Togetherhood

# ======================
# Twilio (SMS)
# ======================
Twilio__AccountSid=your-account-sid
Twilio__AuthToken=your-auth-token
Twilio__PhoneNumber=+1234567890

# ======================
# Tipalti (Payments)
# ======================
Tipalti__ApiKey=your-api-key
Tipalti__ApiUrl=https://ui2.sandbox.tipalti.com/
Tipalti__PayerName=Togetherhood

# ======================
# HubSpot (CRM)
# ======================
HubSpot__ApiKey=your-api-key
HubSpot__PortalId=your-portal-id

# ======================
# Wix API
# ======================
Wix__ApiKey=your-api-key
Wix__SiteId=your-site-id

# ======================
# Google APIs
# ======================
Google__MapsApiKey=your-maps-api-key
Google__PlacesApiKey=your-places-api-key

# ======================
# Application Settings
# ======================
App__AllowedHosts=localhost,*.togetherhood.us
App__EnableSwagger=true
App__MaxUploadSizeBytes=10485760

# ======================
# Logging
# ======================
Logging__LogLevel__Default=Information
Logging__LogLevel__Microsoft=Warning
Logging__LogLevel__Microsoft.EntityFrameworkCore=Warning

# ======================
# Feature Flags (optional)
# ======================
Features__EnableNewDashboard=false
Features__EnableBetaFeatures=false
```

```bash
# apps/web/.env.example
# ======================
# Vite Configuration
# ======================
VITE_API_URL=http://localhost:5000/api
VITE_ENV=development
VITE_PORT=3000

# ======================
# External Services
# ======================
VITE_TIPALTI_ONBOARDING_API=https://ui2.sandbox.tipalti.com/payeedashboard/home
VITE_TIPALTI_API_KEY=your-api-key
VITE_GOOGLE_MAPS_API_KEY=your-maps-api-key

# ======================
# Analytics
# ======================
VITE_DATADOG_CLIENT_TOKEN=your-client-token
VITE_DATADOG_APPLICATION_ID=your-app-id
VITE_HUBSPOT_PORTAL_ID=your-portal-id

# ======================
# Feature Flags
# ======================
VITE_ENABLE_BETA_FEATURES=false
```

---

## 🌿 Branch Strategy & Hotfixes

**Current:**
- `develop` → staging
- `main` → production

**Question:** What about hotfixes?

**Recommendation: GitFlow-style hotfix branch:**

```
main (production)
  ↓
hotfix/fix-critical-bug
  ↓ (PR after testing)
main (deploy to production immediately)
  ↓ (merge back)
develop (update staging with fix)
```

**Workflow:**
1. Critical bug found in production
2. Create `hotfix/` branch from `main`
3. Fix and test locally
4. PR to `main` (expedited review)
5. Merge to `main` → auto-deploy to production
6. Merge `main` back to `develop` to keep in sync

**GitHub Actions:**
```yaml
# .github/workflows/hotfix.yml
name: Hotfix Deploy

on:
  push:
    branches:
      - 'hotfix/**'

jobs:
  validate:
    # Run tests

  deploy-to-staging:
    # Deploy to staging for quick validation

  notify:
    # Notify team that hotfix is ready for production
```

---

## 👥 Team Access & IAM

**Create IAM users/roles for team members:**

```bash
# Developer group (read-only CloudWatch, limited deployment)
aws iam create-group --group-name TogetherhoodDevelopers

aws iam attach-group-policy \
    --group-name TogetherhoodDevelopers \
    --policy-arn arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess

# DevOps group (full access to App Runner, RDS, ECR)
aws iam create-group --group-name TogetherhoodDevOps

# QA group (access to staging only)
aws iam create-group --group-name TogetherhoodQA
```

**GitHub Teams:**
- **Admins:** Can merge to `main`, full access
- **Developers:** Can merge to `develop`, staging access
- **QA:** Read access, can comment on PRs

---

## 📚 Documentation for Team

**Create these docs:**

1. **ONBOARDING.md** - New developer setup
2. **DEPLOYMENT.md** - How to deploy
3. **TROUBLESHOOTING.md** - Common issues
4. **RUNBOOK.md** - On-call procedures
5. **API_DOCUMENTATION.md** - API endpoints
6. **ARCHITECTURE.md** - System architecture

---

## 🧪 Testing in Production

**Considerations:**

1. **Feature Flags** - Gradual rollout
2. **Canary Deployments** - App Runner doesn't support natively, use DNS weighting
3. **A/B Testing** - Use feature flags or CloudFront
4. **Synthetic Monitoring** - CloudWatch Synthetics for continuous testing

---

## 📊 Business Metrics

**Track beyond technical metrics:**

```csharp
// Example custom metrics
- Partners created per day
- Courses enrolled per day
- Active users (DAU/MAU)
- Revenue per day
- API usage by endpoint
- User session duration
```

**Send to CloudWatch:**
```csharp
await cloudWatch.PutMetricDataAsync(new PutMetricDataRequest
{
    Namespace = "Togetherhood/Business",
    MetricData = new List<MetricDatum>
    {
        new MetricDatum
        {
            MetricName = "DailyActiveUsers",
            Value = activeUserCount,
            Unit = StandardUnit.Count,
            Timestamp = DateTime.UtcNow
        }
    }
});
```

---

## Summary: Priority Matrix

| Item | Priority | Complexity | When |
|------|----------|------------|------|
| **Database Migration Strategy** | 🔴 Critical | Medium | Before Phase 1 |
| **S3 Buckets** | 🔴 Critical | Low | Before Phase 1 |
| **Complete Env Vars** | 🔴 Critical | Low | Before Phase 1 |
| **Structured Logging** | 🟡 High | Low | Phase 1 |
| **Comprehensive Health Checks** | 🟡 High | Low | Phase 1 |
| **CloudWatch Dashboards** | 🟡 High | Medium | Phase 1 |
| **Alerting Setup** | 🟡 High | Medium | Phase 1 |
| **CORS Configuration** | 🟡 High | Low | Phase 1 |
| **Cost Budgets** | 🟡 High | Low | Phase 1 |
| **Data Migration Plan** | 🔴 Critical | High | Before Phase 3 |
| **Hotfix Strategy** | 🟡 High | Low | Phase 2 |
| **Team IAM Access** | 🟢 Medium | Low | Phase 2 |
| **VPC Configuration** | 🟢 Medium | High | Phase 2+ |
| **Redis/Caching** | 🟢 Low | Medium | If needed |
| **WAF/Rate Limiting** | 🟢 Low | Medium | Phase 2+ |
| **APM (Datadog/New Relic)** | 🟢 Low | Medium | If needed |
| **Feature Flags** | 🟢 Low | Medium | Future |

---

## Next Steps

1. Review this checklist with your team
2. Prioritize items based on your needs
3. Add high-priority items to migration plan
4. Create issues/tickets for each item
5. Assign owners

**Related Docs:**
- [AWS Infrastructure Setup](aws-infrastructure-setup.md)
- [Staging Environment Setup](staging-environment.md)
- [CI/CD Testing Strategy](cicd-testing.md)
