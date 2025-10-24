# DNS & Custom Domain Configuration Guide

## Overview

This guide covers configuring custom domains for the Togetherhood App Runner services using **Wix DNS management** with a phased rollout strategy.

---

## Rollout Strategy

### Phase 1: Test Staging with Beta Subdomain
**Goal:** Validate staging environment with real DNS
- **Domain:** `beta.togetherhood.us`
- **Points to:** App Runner Staging
- **Duration:** 1-2 weeks (testing/QA)

### Phase 2: Beta on Production
**Goal:** Soft launch with subset of users
- **Domain:** `beta.togetherhood.us`
- **Points to:** App Runner Production
- **Duration:** 1-2 weeks (production validation)

### Phase 3: Full Production Rollover
**Goal:** Move all traffic to new infrastructure
- **Domain:** `app.togetherhood.us` (or `togetherhood.us`)
- **Points to:** App Runner Production
- **Transition:** Gradual DNS cutover from old infrastructure

---

## Prerequisites

- [ ] App Runner staging service created and running
- [ ] App Runner production service created and running
- [ ] Access to Wix account with DNS management permissions
- [ ] Domain `togetherhood.us` managed in Wix
- [ ] AWS CLI configured

---

## Phase 1: Configure Beta → Staging

### Step 1: Associate Custom Domain with App Runner Staging

```bash
# Get staging service ARN
export STAGING_SERVICE_ARN=$(aws apprunner list-services \
    --region us-east-1 \
    --query "ServiceSummaryList[?ServiceName=='togetherhood-app-staging'].ServiceArn" \
    --output text)

# Associate custom domain
aws apprunner associate-custom-domain \
    --service-arn ${STAGING_SERVICE_ARN} \
    --domain-name beta.togetherhood.us \
    --region us-east-1

echo "✅ Custom domain association initiated"
```

### Step 2: Get Validation Records

```bash
# Get certificate validation records
aws apprunner describe-custom-domains \
    --service-arn ${STAGING_SERVICE_ARN} \
    --region us-east-1 > staging-domain-validation.json

# Extract validation records
cat staging-domain-validation.json | jq -r '.CustomDomains[0].CertificateValidationRecords[]'
```

**Example output:**
```json
{
  "Name": "_abc123.beta.togetherhood.us",
  "Type": "CNAME",
  "Value": "_xyz789.acm-validations.aws.",
  "Status": "PENDING_VALIDATION"
}
```

### Step 3: Add DNS Records in Wix

#### Access Wix DNS Management

1. Log in to **Wix** → Go to your **Dashboard**
2. Navigate to **Domains** → Select `togetherhood.us`
3. Click **Manage DNS Records** or **DNS**

#### Add Validation CNAME Records

For **each validation record** from AWS:

1. Click **+ Add Record**
2. Select **CNAME**
3. Fill in:
   - **Host/Name:** `_abc123.beta` (the subdomain part only)
   - **Value/Points to:** `_xyz789.acm-validations.aws.`
   - **TTL:** 3600 (or 1 hour)
4. Click **Save**

**Important:** Some DNS providers require different formats:
- Wix might want just the subdomain: `_abc123.beta`
- Or the full name without root: `_abc123.beta.togetherhood.us`
- **Check Wix's documentation** if unsure

#### Add CNAME for Beta Subdomain

After validation records are added:

1. Click **+ Add Record**
2. Select **CNAME**
3. Fill in:
   - **Host/Name:** `beta`
   - **Value/Points to:** `<staging-service-url>.us-east-1.awsapprunner.com.` (from App Runner)
   - **TTL:** 300 (5 minutes for easier testing/rollback)
4. Click **Save**

**Get App Runner staging URL:**
```bash
export STAGING_URL=$(aws apprunner list-services \
    --region us-east-1 \
    --query "ServiceSummaryList[?ServiceName=='togetherhood-app-staging'].ServiceUrl" \
    --output text)

echo "CNAME value: ${STAGING_URL}"
```

### Step 4: Wait for Certificate Validation

```bash
# Check validation status (repeat every few minutes)
aws apprunner describe-custom-domains \
    --service-arn ${STAGING_SERVICE_ARN} \
    --region us-east-1 \
    --query 'CustomDomains[0].Status'
```

**Possible statuses:**
- `CREATING` - Initial setup
- `CREATE_FAILED` - Check DNS records
- `ACTIVE` - ✅ Ready to use!

**Typical wait time:** 5-30 minutes for DNS propagation + validation

### Step 5: Verify Beta Staging Works

```bash
# Test DNS resolution
dig beta.togetherhood.us

# Test HTTPS endpoint
curl -I https://beta.togetherhood.us/api/health

# Open in browser
open https://beta.togetherhood.us
```

**Expected:**
- DNS resolves to App Runner staging
- SSL certificate valid for `beta.togetherhood.us`
- Application loads correctly

### Step 6: Update Application Configuration (if needed)

If your app checks domain names or has CORS restrictions:

**Update `.env` or Secrets Manager:**
```bash
# Add to staging secrets
aws secretsmanager update-secret \
    --secret-id togetherhood-app/staging/config \
    --secret-string "$(aws secretsmanager get-secret-value \
        --secret-id togetherhood-app/staging/config \
        --query SecretString --output text | \
        jq '. + {"AllowedHosts": "beta.togetherhood.us"}' \
    )" \
    --region us-east-1

# Restart App Runner to pick up changes
aws apprunner update-service \
    --service-arn ${STAGING_SERVICE_ARN} \
    --region us-east-1
```

**For CORS (if needed):**
```csharp
// Program.cs
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.WithOrigins(
            "http://localhost:3000",
            "https://beta.togetherhood.us"  // Add this
        )
        .AllowAnyMethod()
        .AllowAnyHeader()
        .AllowCredentials();
    });
});
```

---

## Phase 2: Move Beta → Production

**Timeline:** After 1-2 weeks of successful staging testing

### Step 1: Associate Domain with Production App Runner

```bash
# Get production service ARN
export PROD_SERVICE_ARN=$(aws apprunner list-services \
    --region us-east-1 \
    --query "ServiceSummaryList[?ServiceName=='togetherhood-app-production'].ServiceArn" \
    --output text)

# Associate beta domain with production
aws apprunner associate-custom-domain \
    --service-arn ${PROD_SERVICE_ARN} \
    --domain-name beta.togetherhood.us \
    --region us-east-1
```

### Step 2: Get New Validation Records

```bash
# Get production validation records
aws apprunner describe-custom-domains \
    --service-arn ${PROD_SERVICE_ARN} \
    --region us-east-1 > production-domain-validation.json

# Extract validation records
cat production-domain-validation.json | jq -r '.CustomDomains[0].CertificateValidationRecords[]'
```

### Step 3: Update DNS in Wix

#### Remove Old Validation Records (from staging)

1. Go to Wix DNS management
2. Find old validation CNAME records (starting with `_abc123.beta`)
3. Delete them

#### Add New Validation Records (for production)

1. Add new validation CNAME records from production
2. Format: Same as Phase 1

#### Update Beta CNAME to Production

1. Find existing CNAME record: `beta.togetherhood.us`
2. Click **Edit**
3. Update **Value/Points to:** `<production-service-url>.us-east-1.awsapprunner.com.`
4. Click **Save**

**Get production URL:**
```bash
export PROD_URL=$(aws apprunner list-services \
    --region us-east-1 \
    --query "ServiceSummaryList[?ServiceName=='togetherhood-app-production'].ServiceUrl" \
    --output text)

echo "Update CNAME to: ${PROD_URL}"
```

### Step 4: Remove Domain from Staging

```bash
# Disassociate domain from staging (now points to production)
aws apprunner disassociate-custom-domain \
    --service-arn ${STAGING_SERVICE_ARN} \
    --domain-name beta.togetherhood.us \
    --region us-east-1
```

### Step 5: Wait and Verify

```bash
# Wait for validation (5-30 minutes)
watch -n 30 "aws apprunner describe-custom-domains \
    --service-arn ${PROD_SERVICE_ARN} \
    --region us-east-1 \
    --query 'CustomDomains[0].Status'"

# Test DNS resolution (may take time for cache to clear)
dig beta.togetherhood.us

# Verify SSL and application
curl -I https://beta.togetherhood.us/api/health
```

### Step 6: Monitor Production via Beta

**Monitoring checklist:**
- [ ] DNS resolves correctly
- [ ] SSL certificate valid
- [ ] Application loads
- [ ] User login works
- [ ] API endpoints respond
- [ ] Database connections work
- [ ] External integrations work (SendGrid, Twilio, etc.)
- [ ] No errors in CloudWatch logs

**Monitor for 1-2 weeks** before full rollover.

---

## Phase 3: Full Production Rollover

**Goal:** Move primary domain to new App Runner infrastructure

### Option A: New Subdomain (Recommended for testing)

If you want to keep old infrastructure running:

**New domain:** `app.togetherhood.us`
**Old domain:** `togetherhood.us` (stays on old infrastructure)

#### Steps:

```bash
# Associate app subdomain with production
aws apprunner associate-custom-domain \
    --service-arn ${PROD_SERVICE_ARN} \
    --domain-name app.togetherhood.us \
    --region us-east-1

# Get validation records
aws apprunner describe-custom-domains \
    --service-arn ${PROD_SERVICE_ARN} \
    --region us-east-1
```

**In Wix:**
1. Add validation CNAME records
2. Add CNAME: `app` → `<prod-service-url>.awsapprunner.com`

**Gradual migration:**
- Update links in emails/docs to `app.togetherhood.us`
- Add redirect from `togetherhood.us` → `app.togetherhood.us`
- Monitor traffic shift
- After 100% cutover, deprecate old infrastructure

### Option B: Root Domain Cutover (Riskier)

If you want to use `togetherhood.us` (root/apex domain):

**⚠️ Limitation:** App Runner requires CNAME, but root domains can't use CNAME.

**Solutions:**

#### Solution 1: Use ALIAS record (if Wix supports)

Some DNS providers (like Route 53) support ALIAS records for apex domains.

**Check if Wix supports ALIAS/ANAME records:**
- Look for "ALIAS" or "ANAME" record type in Wix DNS
- If available, create ALIAS for `@` (root) → App Runner URL

#### Solution 2: Use Subdomain Redirect

**Keep:** `togetherhood.us` with redirect
**Use:** `app.togetherhood.us` for actual app

**In Wix:**
1. Keep existing A record for `togetherhood.us`
2. Set up HTTP 301 redirect: `togetherhood.us` → `app.togetherhood.us`
3. Users always redirected to subdomain

#### Solution 3: CloudFront + App Runner (Future)

For true apex domain support:

```
togetherhood.us (A record to CloudFront)
    ↓
CloudFront Distribution
    ↓
App Runner (origin)
```

**Benefits:**
- Apex domain support
- CDN caching
- Better performance
- DDoS protection

**Drawback:** Adds complexity

### Recommended Approach

Use **`app.togetherhood.us`** for App Runner and redirect from root:

```
togetherhood.us → app.togetherhood.us (301 redirect)
www.togetherhood.us → app.togetherhood.us (301 redirect)
beta.togetherhood.us → Production (for testing)
```

---

## Wix DNS Configuration Reference

### Record Types Used

| Type | Purpose | Example |
|------|---------|---------|
| **CNAME** | Point subdomain to App Runner | `beta` → `xyz.awsapprunner.com` |
| **CNAME** | Certificate validation | `_abc.beta` → `_xyz.acm-validations.aws` |
| **A** | Root domain (if needed) | `@` → CloudFront IP |
| **TXT** | Domain verification | `@` → verification string |

### Common Wix DNS Formats

**Subdomain CNAME:**
```
Type: CNAME
Host: beta
Points to: abc123.us-east-1.awsapprunner.com
TTL: 300
```

**Validation CNAME:**
```
Type: CNAME
Host: _abc123.beta
Points to: _xyz789.acm-validations.aws.
TTL: 3600
```

**Notes:**
- Wix might auto-append `.togetherhood.us` to host names
- Always include trailing `.` for CNAME targets (some systems require it)
- Use lowercase for consistency

---

## DNS Propagation & Testing

### Check DNS Propagation

```bash
# Check from your location
dig beta.togetherhood.us
nslookup beta.togetherhood.us

# Check from multiple locations (online tool)
# https://www.whatsmydns.net/#CNAME/beta.togetherhood.us

# Check with specific DNS server
dig @8.8.8.8 beta.togetherhood.us
dig @1.1.1.1 beta.togetherhood.us
```

### DNS Propagation Timeline

| Stage | Time | Notes |
|-------|------|-------|
| **Wix DNS update** | Immediate | Record saved in Wix |
| **Nameserver propagation** | 5-15 min | Wix nameservers update |
| **Global propagation** | 1-24 hours | ISP DNS caches (depends on TTL) |
| **AWS validation** | 5-30 min | After DNS visible to AWS |
| **SSL certificate** | 5-10 min | After validation complete |

**Speed up propagation:**
- Set low TTL (300s) before making changes
- Use `dig` with specific DNS servers to check
- Clear local DNS cache: `sudo dscacheutil -flushcache` (macOS)

---

## SSL Certificate Management

### Certificate Validation

App Runner automatically provisions SSL certificates using AWS Certificate Manager (ACM) when you associate a custom domain.

**Validation steps:**
1. You add domain to App Runner
2. App Runner provides DNS validation records
3. You add CNAME records to Wix DNS
4. AWS validates domain ownership (checks CNAME)
5. ACM issues SSL certificate
6. App Runner serves HTTPS traffic

**Certificate auto-renewal:**
- ACM automatically renews certificates before expiry
- Uses same DNS validation records
- No manual intervention needed

### Verify SSL Certificate

```bash
# Check certificate details
openssl s_client -connect beta.togetherhood.us:443 -servername beta.togetherhood.us < /dev/null

# Check expiry
echo | openssl s_client -servername beta.togetherhood.us -connect beta.togetherhood.us:443 2>/dev/null | openssl x509 -noout -dates

# Browser check
open https://beta.togetherhood.us
# Click padlock → View certificate
```

---

## Rollback Procedures

### Rollback During Phase 1 (Beta → Staging)

If issues found:

```bash
# Option 1: Point beta back to old infrastructure
# Update CNAME in Wix to old server

# Option 2: Remove custom domain entirely
aws apprunner disassociate-custom-domain \
    --service-arn ${STAGING_SERVICE_ARN} \
    --domain-name beta.togetherhood.us \
    --region us-east-1

# Remove DNS records in Wix
```

### Rollback During Phase 2 (Beta → Production)

If production issues found:

```bash
# Point beta back to staging
# 1. Reassociate with staging
aws apprunner associate-custom-domain \
    --service-arn ${STAGING_SERVICE_ARN} \
    --domain-name beta.togetherhood.us \
    --region us-east-1

# 2. Update CNAME in Wix to staging URL
# 3. Wait for DNS propagation (5-15 minutes)
```

### Rollback During Phase 3 (Full Rollover)

If major issues after full rollover:

```bash
# Update DNS in Wix to point back to old infrastructure
# Change CNAME or A records to old server IPs
# Wait for DNS propagation
```

**DNS TTL consideration:**
- Low TTL (300s) = Faster rollback, more DNS queries
- High TTL (3600s) = Slower rollback, fewer DNS queries

**Recommendation:** Use 300s TTL during migration, increase to 3600s after stable.

---

## Monitoring & Validation

### Health Checks

```bash
# Monitor App Runner health
aws apprunner describe-service \
    --service-arn ${PROD_SERVICE_ARN} \
    --region us-east-1 \
    --query 'Service.Status'

# Check custom domain status
aws apprunner describe-custom-domains \
    --service-arn ${PROD_SERVICE_ARN} \
    --region us-east-1 \
    --query 'CustomDomains[0].Status'
```

### CloudWatch Alarms

Create alarms for custom domain issues:

```bash
# Alarm for 4xx errors
aws cloudwatch put-metric-alarm \
    --alarm-name beta-4xx-errors \
    --alarm-description "Beta domain returning 4xx errors" \
    --metric-name 4xxStatusResponses \
    --namespace AWS/AppRunner \
    --statistic Sum \
    --period 300 \
    --threshold 50 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --dimensions Name=ServiceName,Value=togetherhood-app-production

# Alarm for 5xx errors
aws cloudwatch put-metric-alarm \
    --alarm-name beta-5xx-errors \
    --alarm-description "Beta domain returning 5xx errors" \
    --metric-name 5xxStatusResponses \
    --namespace AWS/AppRunner \
    --statistic Sum \
    --period 300 \
    --threshold 10 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --dimensions Name=ServiceName,Value=togetherhood-app-production
```

### Application Monitoring

```bash
# Tail logs
aws logs tail /aws/apprunner/togetherhood-app-production/application \
    --follow \
    --filter-pattern "ERROR" \
    --region us-east-1

# Check recent deployments
aws apprunner list-operations \
    --service-arn ${PROD_SERVICE_ARN} \
    --region us-east-1 \
    --max-results 5
```

---

## Troubleshooting

### DNS Not Resolving

**Symptoms:** `dig beta.togetherhood.us` returns NXDOMAIN

**Solutions:**
1. Verify CNAME record added in Wix
2. Check record format (host: `beta`, not `beta.togetherhood.us`)
3. Wait 5-15 minutes for propagation
4. Check Wix nameservers: `dig NS togetherhood.us`
5. Verify domain not paused/suspended in Wix

### SSL Certificate Validation Fails

**Symptoms:** `CustomDomain` status stuck in `CREATING` or `CREATE_FAILED`

**Solutions:**
1. Verify validation CNAME records added correctly
2. Check record exactly matches AWS output (including `.` at end)
3. Wait 30 minutes for DNS propagation
4. Remove and re-add custom domain in App Runner
5. Check CloudWatch logs for validation errors

### Application Not Loading

**Symptoms:** DNS resolves but HTTPS gives error

**Solutions:**
1. Check App Runner service is in `RUNNING` state
2. Verify health check endpoint works: `/api/health`
3. Check security groups allow inbound HTTPS
4. Review CloudWatch logs for application errors
5. Verify database connection string correct
6. Check Secrets Manager configuration

### Mixed Content Warnings

**Symptoms:** Browser shows "Not Secure" despite HTTPS

**Solutions:**
1. Ensure all API calls use HTTPS
2. Check for hardcoded HTTP URLs in frontend
3. Update `VITE_API_URL` to use HTTPS
4. Add `Strict-Transport-Security` header

---

## Best Practices

### ✅ DO

- ✅ Use low TTL (300s) during migration
- ✅ Test thoroughly on beta before production rollover
- ✅ Monitor closely for 24-48 hours after DNS changes
- ✅ Keep old infrastructure running during beta testing
- ✅ Document all DNS changes and timing
- ✅ Use subdomain (`app.togetherhood.us`) over apex for App Runner
- ✅ Set up CloudWatch alarms before DNS cutover

### ❌ DON'T

- ❌ Rush DNS changes in production
- ❌ Change multiple DNS records simultaneously
- ❌ Ignore DNS propagation times
- ❌ Delete old infrastructure immediately after cutover
- ❌ Use high TTL during migration (delays rollback)
- ❌ Skip testing SSL certificates after DNS changes
- ❌ Forget to update application CORS/allowed hosts

---

## Migration Timeline

### Recommended Timeline

| Week | Phase | Domain | Action |
|------|-------|--------|--------|
| **Week 1** | Beta Staging | `beta.togetherhood.us` → Staging | Add DNS, test QA |
| **Week 2** | Beta Staging | `beta.togetherhood.us` → Staging | Continue testing |
| **Week 3** | Beta Production | `beta.togetherhood.us` → Production | Move to production |
| **Week 4-5** | Beta Production | `beta.togetherhood.us` → Production | Monitor, validate |
| **Week 6** | Full Rollover | `app.togetherhood.us` → Production | Add new domain |
| **Week 7** | Redirect | `togetherhood.us` → `app.togetherhood.us` | Redirect root |
| **Week 8** | Cleanup | - | Remove old infrastructure |

### Accelerated Timeline (if confident)

| Week | Phase | Action |
|------|-------|--------|
| **Week 1** | Beta Staging | Test staging |
| **Week 2** | Beta Production | Move to prod, monitor |
| **Week 3** | Full Rollover | Add app subdomain, redirect root |
| **Week 4** | Cleanup | Remove old infrastructure |

---

## Summary Checklist

### Phase 1: Beta → Staging
- [ ] Associate `beta.togetherhood.us` with App Runner Staging
- [ ] Get validation records from AWS
- [ ] Add validation CNAMEs to Wix
- [ ] Add beta CNAME to Wix
- [ ] Wait for validation (5-30 min)
- [ ] Test HTTPS and application
- [ ] QA testing for 1-2 weeks

### Phase 2: Beta → Production
- [ ] Associate `beta.togetherhood.us` with App Runner Production
- [ ] Get new validation records
- [ ] Update validation CNAMEs in Wix
- [ ] Update beta CNAME to production URL
- [ ] Remove domain from staging App Runner
- [ ] Wait for validation
- [ ] Monitor production via beta for 1-2 weeks

### Phase 3: Full Rollover
- [ ] Decide: `app.togetherhood.us` or apex domain
- [ ] Associate production domain with App Runner
- [ ] Add DNS records to Wix
- [ ] Set up root domain redirect (if needed)
- [ ] Update application configuration
- [ ] Monitor for 1 week
- [ ] Deprecate old infrastructure

---

## Next Steps

1. ✅ Complete App Runner setup (staging + production)
2. ✅ Start with Phase 1: Beta → Staging
3. ✅ Test thoroughly for 1-2 weeks
4. ✅ Proceed to Phase 2 when confident
5. ✅ Plan full rollover timeline with stakeholders

**Related Guides:**
- [AWS Infrastructure Setup](aws-infrastructure-setup.md)
- [Staging Environment Setup](staging-environment.md)
- [Migration Checklist](migration-checklist.md) - See Rollback Strategy section
- [CI/CD Testing Strategy](cicd-testing.md)
