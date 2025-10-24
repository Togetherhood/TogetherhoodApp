# TogetherhoodApp Migration Documentation

**Project Beowulf - Simplified Deployment Phase**

This documentation provides everything you need to migrate TogetherhoodWEB and TogetherhoodAPI into a unified monorepo with AWS App Runner deployment.

---

## 📖 Quick Navigation

### Start Here

**[📋 Complete Migration Plan](migration-plan.md)** - Detailed 8-phase step-by-step implementation guide

This is your main resource. It contains:
- Complete phase-by-phase instructions
- All bash commands and code examples
- Docker configurations
- Git history preservation steps
- Local development setup
- AWS infrastructure setup
- CI/CD configuration

### Implementation Guides (Topic-Specific Deep Dives)

Use these guides for detailed information on specific topics:

- **[⚠️ Migration Checklist](guides/migration-checklist.md)** - Critical items to review BEFORE starting
- **[AWS Infrastructure Setup](guides/aws-infrastructure-setup.md)** - ECR, App Runner (staging + prod), RDS configuration
- **[AWS Secrets Manager](guides/aws-secrets-manager.md)** - Secrets management for staging and production
- **[Staging Environment](guides/staging-environment.md)** - Complete staging environment setup
- **[DNS & Custom Domains](guides/dns-custom-domains.md)** - 3-phase rollout strategy with Wix DNS
- **[CI/CD Testing Strategy](guides/cicd-testing.md)** - Unit tests, vitest, dotnet test, E2E tests
- **[E2E Testing Integration](guides/e2e-testing.md)** - WebDriverIO + Cucumber setup

---

## 🚀 Overview

### What This Migration Does

Consolidates two separate repositories into one unified monorepo:

| Before | After |
|--------|-------|
| **2 repositories** (TogetherhoodWEB, TogetherhoodAPI) | **1 monorepo** (TogetherhoodApp) |
| **2 deployment pipelines** to Elastic Beanstalk | **1 pipeline** to AWS App Runner |
| **2 Docker images** (or none) | **1 multi-stage image** |
| **Run 2 environments** separately | **`docker-compose up`** - everything runs |
| **Separate git histories** | **Merged history** - fully preserved |

### Key Benefits

✅ **Atomic deployments** - Frontend + backend always in sync
✅ **Simplified CI/CD** - One pipeline, one artifact
✅ **Better developer experience** - One checkout, one command
✅ **No CORS complexity** - Same origin for API and SPA
✅ **Reduced operational overhead** - One service to monitor

### Technology Stack

| Layer | Technology |
|-------|------------|
| **Frontend** | React 19 + Vite + TypeScript |
| **Backend** | .NET 6 Web API |
| **Build** | Docker Multi-Stage |
| **Runtime** | AWS App Runner (staging + production) |
| **CI/CD** | GitHub Actions |
| **Database** | MySQL 8.0 RDS (staging + production) |
| **Secrets** | AWS Secrets Manager |

---

## 📂 Target Monorepo Structure

```
TogetherhoodApp/
├── apps/
│   ├── api/          # .NET 6 Web API
│   ├── web/          # React 19 + Vite + TypeScript
│   └── e2e/          # E2E tests (WebDriverIO + Cucumber)
├── docker/
│   ├── Dockerfile                # Multi-stage production build
│   └── docker-compose.yml        # Local development
├── .github/workflows/            # CI/CD pipelines
├── docs/                         # This documentation
└── scripts/                      # Utility scripts
```

---

## ⏱️ Migration Timeline

| Phase | Duration | What You'll Do |
|-------|----------|----------------|
| **Phase 1** | 1-2 hours | Git history preservation & repo setup |
| **Phase 2** | 1 hour | Monorepo structure setup |
| **Phase 3** | 2 hours | Docker production configuration |
| **Phase 4** | 2 hours | Local development setup |
| **Phase 5** | 2-3 hours | AWS infrastructure setup |
| **Phase 6** | 1-2 hours | CI/CD pipeline configuration |
| **Phase 7** | 2-3 hours | Testing & validation |
| **Phase 8** | 1-2 hours | Production cutover |
| **Total** | **1-2 days** | With testing buffer |

---

## 🚦 How to Use This Documentation

### For Migration Team

1. **Start here:** Review the [Migration Checklist](guides/migration-checklist.md) for critical considerations
2. **Follow the plan:** Work through the [Complete Migration Plan](migration-plan.md) phase by phase
3. **Deep dive when needed:** Reference the implementation guides for detailed topic-specific information

### For Developers (Post-Migration)

After migration is complete:

```bash
# Clone the monorepo
git clone https://github.com/Togetherhood/TogetherhoodApp.git
cd TogetherhoodApp

# Start local development
docker-compose -f docker/docker-compose.yml up

# Access:
# Web: http://localhost:3000
# API: http://localhost:5000
# DB: localhost:3306
```

### For DevOps/Infrastructure

Focus on these guides:
1. [AWS Infrastructure Setup](guides/aws-infrastructure-setup.md)
2. [AWS Secrets Manager](guides/aws-secrets-manager.md)
3. [Staging Environment](guides/staging-environment.md)
4. [CI/CD Testing Strategy](guides/cicd-testing.md)

---

## 📝 Getting Started

**👉 Go to:** **[Complete Migration Plan](migration-plan.md)** and start with Phase 1.

---

## 📊 Success Metrics

After migration, you should have:

- ✅ Single `docker-compose up` starts complete dev environment
- ✅ Single git repository with full history from both repos
- ✅ Single CI/CD pipeline builds and deploys to production
- ✅ Deployment time reduced from ~15 minutes to ~5 minutes
- ✅ Simplified infrastructure (1 App Runner service per environment vs multiple EB environments)
- ✅ Better developer experience (faster onboarding, simpler workflow)

---

## 🆘 Support

For troubleshooting, see the **Troubleshooting** section in the [Migration Checklist](guides/migration-checklist.md).

---

**Ready to begin?** → **[Start with the Complete Migration Plan](migration-plan.md)**
