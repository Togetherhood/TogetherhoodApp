# TogetherhoodApp

**Unified monorepo for Togetherhood Web + API**

## 📚 Documentation

This repository contains the complete migration plan and documentation for consolidating TogetherhoodWEB and TogetherhoodAPI into a unified monorepo with simplified Docker deployment to AWS App Runner.

**[📖 View Complete Documentation](docs/README.md)**

## 🚀 Project Beowulf - Simplified Deployment Phase

### Current Status

**Phase:** Planning & Documentation
**Version:** 1.0
**Last Updated:** October 2025

This repository is currently in the planning phase. The documentation is being reviewed by the team before beginning the actual migration work.

### What's Inside

**📚 Documentation:**
- **[docs/README.md](docs/README.md)** - Documentation hub (start here)
- **[docs/migration-plan.md](docs/migration-plan.md)** - Complete 8-phase step-by-step guide
- **[docs/guides/](docs/guides/)** - Topic-specific implementation guides

**Key Guides:**
- [Migration Checklist](docs/guides/migration-checklist.md) - Critical items (READ FIRST)
- [AWS Infrastructure](docs/guides/aws-infrastructure-setup.md) - ECR, App Runner, RDS
- [AWS Secrets Manager](docs/guides/aws-secrets-manager.md) - Secrets management
- [Staging Environment](docs/guides/staging-environment.md) - Staging setup
- [DNS & Custom Domains](docs/guides/dns-custom-domains.md) - 3-phase rollout
- [CI/CD Testing](docs/guides/cicd-testing.md) - Testing strategy
- [E2E Testing](docs/guides/e2e-testing.md) - WebDriverIO + Cucumber

### Target Architecture

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
└── docs/                         # This documentation
```

### Key Benefits

✅ **Atomic deployments** - Frontend + backend always in sync
✅ **Simplified CI/CD** - One pipeline, one artifact
✅ **Better developer experience** - One checkout, one command
✅ **No CORS complexity** - Same origin for API and SPA
✅ **Reduced operational overhead** - One service to monitor

### Technology Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Frontend** | React 19 + Vite + TypeScript | Modern SPA with hot reload |
| **Backend** | .NET 6 Web API | REST API + static file serving |
| **Build** | Docker Multi-Stage | Single deployable artifact |
| **Runtime** | AWS App Runner | Managed container platform |
| **Environments** | Staging + Production | Separate App Runner services |
| **CI/CD** | GitHub Actions | Automated build + deploy |
| **Database** | MySQL 8.0 RDS | Persistent data (staging + prod) |

### Quick Start (After Migration)

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

## 🤝 Contributing

This repository is currently in planning phase. Once the migration is complete, developers should:

1. Clone this monorepo
2. Create short-lived feature branches from `main`
3. Submit PRs for review (keep them small!)
4. Merges to `main` trigger CI/CD
5. `main` is always deployable and auto-deploys to production
6. Use feature flags for incomplete features

**Trunk-based development:** We work directly on `main` with short-lived branches. No `develop` branch.

## 📞 Support

For questions about the migration plan or this documentation, please review the guides in the `docs/` directory or reach out to the DevOps team.

---

**Status:** 🟡 In Planning - Documentation Review Phase
