# TogetherhoodApp Monorepo Migration Plan
## Project Beowulf - Simplified Deployment Phase

**Version:** 1.0
**Date:** October 2025
**Status:** Ready for Implementation

This document contains the complete step-by-step migration plan for consolidating TogetherhoodWEB and TogetherhoodAPI into a unified monorepo.

---

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [Architecture Overview](#architecture-overview)
3. [Prerequisites](#prerequisites)
4. [Migration Phases](#migration-phases)
5. [Detailed Implementation](#detailed-implementation)
6. [AWS Infrastructure Setup](#aws-infrastructure-setup)
7. [Testing & Validation](#testing--validation)
8. [Rollback Strategy](#rollback-strategy)
9. [Post-Migration Checklist](#post-migration-checklist)

---

## Executive Summary

### Current State
- **TogetherhoodWEB**: React 19 + Vite frontend in separate repository
- **TogetherhoodAPI**: .NET 6 Web API in separate repository
- **Deployment**: Separate CI/CD pipelines to AWS Elastic Beanstalk
- **Dev Experience**: Requires running two separate development environments

### Target State
- **Single Repository**: `Togetherhood/TogetherhoodApp` monorepo
- **Single Docker Image**: Multi-stage build containing both frontend and backend
- **Single Deployment**: One container to AWS App Runner
- **Single Command Dev**: `docker-compose up` for complete local environment
- **Preserved History**: Complete git history from both repositories maintained

### Key Benefits
✅ Atomic deployments (frontend + backend always in sync)
✅ Simplified CI/CD (one pipeline, one artifact)
✅ Better developer experience (one checkout, one command)
✅ No CORS complexity (same origin for API and SPA)
✅ Reduced operational overhead (one service to monitor)
✅ Easier to maintain version consistency

### Migration Timeline
- **Phase 1-2**: 2-3 hours (Repository setup)
- **Phase 3-4**: 2-3 hours (Docker configuration)
- **Phase 5-6**: 2-4 hours (AWS + CI/CD setup)
- **Phase 7-8**: 2-3 hours (Testing + deployment)
- **Total**: ~1-2 days (with buffer for testing)

---

## Architecture Overview

### Technology Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Frontend** | React 19 + Vite + TypeScript | Modern SPA with hot reload |
| **Backend** | .NET 6 Web API | REST API + static file serving |
| **Build** | Docker Multi-Stage | Single deployable artifact |
| **Runtime** | AWS App Runner | Managed container platform |
| **CI/CD** | GitHub Actions | Automated build + deploy |
| **Registry** | AWS ECR | Container image storage |
| **Database** | MySQL 8.0 | Persistent data (RDS/existing) |

### Request Flow (Production)

```
User Browser
    ↓ HTTPS
AWS App Runner (Port 8080)
    ↓
.NET Kestrel Server
    ├─→ /api/* → API Controllers (C#)
    └─→ /* → Static Files (Vite build in wwwroot/)
```

### Development Flow (Local)

```
Developer Machine
    ↓ docker-compose up
┌─────────────────────────────────────────┐
│  MySQL Container (Port 3306)             │
│  API Container (Port 5000)               │
│  Web Dev Server (Port 3000, Vite HMR)   │
└─────────────────────────────────────────┘
    ↓
Browser: http://localhost:3000
    ↓ /api/* proxied to → http://localhost:5000
```

### Monorepo Structure

```
TogetherhoodApp/
├── .github/
│   └── workflows/
│       ├── ci-pr.yml                    # PR validation
|       ├── claude-etc.yml               # Placeholder for Claude Actions
│       └── cd-production.yml            # Deploy to App Runner
├── apps/
│   ├── api/                             # .NET 6 Web API
│   │   ├── Togetherhood.API/
│   │   ├── Togetherhood.Application/
│   │   ├── Togetherhood.Data/
│   │   ├── Togetherhood.Libraries/
│   │   ├── *.Tests/
│   │   └── Togetherhood.sln
│   └── web/                             # React + Vite SPA
│       ├── src/
│       ├── public/
│       ├── package.json
│       ├── vite.config.ts
│       └── tsconfig.json
├── docker/
│   ├── Dockerfile                       # Multi-stage production build
│   ├── docker-compose.yml               # Local development setup
│   └── .dockerignore
├── docs/
│   ├── ARCHITECTURE.md
│   ├── DEVELOPMENT.md
│   └── DEPLOYMENT.md
├── scripts/
│   ├── local-setup.sh                   # Initialize local environment
│   └── health-check.sh                  # Verify deployment
├── .gitignore
├── README.md
└── MONOREPO_MIGRATION_PLAN.md          # This document
```

---

## Prerequisites

### Tools Required
- [ ] **Git** 2.30+ (for subtree merge)
- [ ] **Docker Desktop** 4.0+ (for local development)
- [ ] **Docker Compose** 2.0+
- [ ] **.NET 6 SDK** (for local API development)
- [ ] **Node.js** 20+ (for local web development)
- [ ] **AWS CLI** 2.0+ (configured with credentials)
- [ ] **GitHub CLI** (optional, for repo creation)

### AWS Access Requirements
- [ ] AWS Account with permissions for:
  - ECR (Create repositories, push images)
  - App Runner (Create/manage services)
  - IAM (Create roles for App Runner)
  - Secrets Manager or Parameter Store (for environment variables)
- [ ] AWS Access Key ID and Secret Access Key for GitHub Actions

### GitHub Requirements
- [ ] Organization admin access to `Togetherhood`
- [ ] Ability to create new repositories
- [ ] Ability to configure repository secrets

### Knowledge Prerequisites
- [ ] Basic understanding of Docker and containers
- [ ] Familiarity with Git branching and merging
- [ ] Understanding of .NET and Node.js project structures
- [ ] AWS basic concepts (IAM, ECR, container services)

---

## Migration Phases

### Phase 1: Git History Preservation & Repository Setup
**Duration:** 1-2 hours
**Goal:** Create new monorepo while preserving complete git history

**Steps:**
1. Create new GitHub repository
2. Merge TogetherhoodAPI history into `/apps/api`
3. Merge TogetherhoodWEB history into `/apps/web`
4. Verify history preservation

### Phase 2: Monorepo Structure Setup
**Duration:** 1 hour
**Goal:** Organize code into monorepo structure and update configurations

**Steps:**
1. Update .NET solution file paths
2. Update Vite configuration
3. Create root-level documentation
4. Update .gitignore files

### Phase 3: Docker Production Configuration
**Duration:** 2 hours
**Goal:** Create multi-stage Dockerfile for production deployment

**Steps:**
1. Create multi-stage Dockerfile
2. Configure .NET to serve static files
3. Test Docker build locally
4. Optimize image size and build time

### Phase 4: Local Development Setup
**Duration:** 2 hours
**Goal:** Enable full-stack local development with single command

**Steps:**
1. Create docker-compose.yml
2. Configure Vite dev server with API proxy
3. Setup MySQL initialization scripts
4. Test complete local stack

### Phase 5: AWS Infrastructure Setup
**Duration:** 2-3 hours
**Goal:** Provision AWS resources for container deployment

**Steps:**
1. Create ECR repository
2. Create App Runner service
3. Configure IAM roles and policies
4. Setup environment variables / secrets

### Phase 6: CI/CD Pipeline Configuration
**Duration:** 1-2 hours
**Goal:** Automate build and deployment process

**Steps:**
1. Create GitHub Actions workflow
2. Configure repository secrets
3. Setup branch protection rules
4. Test CI/CD pipeline

### Phase 7: Testing & Validation
**Duration:** 2-3 hours
**Goal:** Ensure everything works end-to-end

**Steps:**
1. Test local development workflow
2. Test Docker build and run
3. Test CI/CD pipeline
4. Validate production deployment

### Phase 8: Production Cutover
**Duration:** 1-2 hours
**Goal:** Switch production traffic to new deployment

**Steps:**
1. Deploy to production App Runner
2. Update DNS/routing if needed
3. Monitor application health
4. Deprecate old deployments

---

## Detailed Implementation

### Phase 1: Git History Preservation & Repository Setup

#### Step 1.1: Create New Repository

```bash
# Navigate to parent directory
cd /home/joe-burns/Projects/togetherhood

# Create new directory for monorepo
mkdir TogetherhoodApp
cd TogetherhoodApp

# Initialize new git repository
git init
git checkout -b main

# Create initial commit
echo "# TogetherhoodApp" > README.md
git add README.md
git commit -m "Initial commit"

# Create repository on GitHub (using gh CLI)
gh repo create Togetherhood/TogetherhoodApp --public --source=. --remote=origin

# Or manually create on GitHub and add remote
git remote add origin https://github.com/Togetherhood/TogetherhoodApp.git
```

#### Step 1.2: Merge TogetherhoodAPI with History Preservation

```bash
# Add TogetherhoodAPI as a remote
git remote add api-source ../TogetherhoodAPI
git fetch api-source

# Create a branch from API's history
git checkout -b merge-api api-source/develop
# or api-source/main depending on your default branch

# Move all files to apps/api subdirectory
mkdir -p apps/api
git ls-tree -z --name-only HEAD | xargs -0 -I {} git mv {} apps/api/

# Commit the move
git commit -m "Move TogetherhoodAPI to apps/api"

# Switch back to main and merge
git checkout main
git merge merge-api --allow-unrelated-histories -m "Merge TogetherhoodAPI history into apps/api"

# Remove temporary remote
git remote remove api-source
```

#### Step 1.3: Merge TogetherhoodWEB with History Preservation

```bash
# Add TogetherhoodWEB as a remote
git remote add web-source ../TogetherhoodWEB
git fetch web-source

# Create a branch from WEB's history
git checkout -b merge-web web-source/develop
# or web-source/main depending on your default branch

# Move all files to apps/web subdirectory
mkdir -p apps/web
git ls-tree -z --name-only HEAD | xargs -0 -I {} git mv {} apps/web/

# Commit the move
git commit -m "Move TogetherhoodWEB to apps/web"

# Switch back to main and merge
git checkout main
git merge merge-web --allow-unrelated-histories -m "Merge TogetherhoodWEB history into apps/web"

# Remove temporary remote
git remote remove web-source

# Clean up branches
git branch -D merge-api merge-web
```

#### Step 1.4: Verify History Preservation

```bash
# Check API history
git log --oneline apps/api/ | head -20

# Check WEB history
git log --oneline apps/web/ | head -20

# Verify all commits are preserved
git log --all --graph --oneline | head -50
```

---

### Phase 2: Monorepo Structure Setup

#### Step 2.1: Create Root Directory Structure

```bash
# Create standard monorepo directories
mkdir -p docker docs scripts

# Create placeholder files
touch docker/.gitkeep
touch docs/.gitkeep
touch scripts/.gitkeep

git add .
git commit -m "Setup: Add monorepo directory structure"
```

#### Step 2.2: Update .NET Solution Paths

The .NET solution file needs to be updated to reflect new paths. However, since projects are already in `apps/api/`, the solution should continue to work as-is when opened from `apps/api/`.

**Verification:**
```bash
# Test that solution still builds
cd apps/api
dotnet build Togetherhood.sln
cd ../..
```

If there are path issues, update the `.sln` file manually or regenerate it.

#### Step 2.3: Update Vite Configuration

Edit `apps/web/vite.config.ts` to add API proxy for local development:

```typescript
import { defineConfig, loadEnv } from 'vite'
import path from 'path'
import react from "@vitejs/plugin-react";
import svgr from "vite-plugin-svgr";
import dts from "vite-plugin-dts";

// https://vitejs.dev/config/
export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd());

  return {
    plugins: [react(), svgr(), dts()],
    resolve: {
      alias: {
        '@': path.resolve(__dirname, './src'),
      },
    },
    server: {
      host: true,
      port: parseInt(env.VITE_PORT || "3000", 10),
      proxy: {
        '/api': {
          target: env.VITE_API_URL || 'http://localhost:5000',
          changeOrigin: true,
          secure: false,
        }
      }
    },
  }
});
```

#### Step 2.4: Create Root-Level Documentation

Create `README.md` in the root:

```markdown
# TogetherhoodApp

Unified monorepo for Togetherhood's web application and API.

## Quick Start

### Local Development
```bash
# Start complete development environment (API + Web + MySQL)
docker-compose -f docker/docker-compose.yml up

# Access the application
# Web: http://localhost:3000
# API: http://localhost:5000
# Database: localhost:3306
```

### Production Build
```bash
# Build Docker image
docker build -f docker/Dockerfile -t togetherhood-app .

# Run production container
docker run -p 8080:8080 togetherhood-app
```

## Project Structure
- `apps/api/` - .NET 6 Web API
- `apps/web/` - React + Vite frontend
- `docker/` - Docker configurations
- `docs/` - Documentation
- `scripts/` - Utility scripts

## Documentation
- [Architecture](docs/ARCHITECTURE.md)
- [Development Guide](docs/DEVELOPMENT.md)
- [Deployment Guide](docs/DEPLOYMENT.md)

## Tech Stack
- Frontend: React 19, TypeScript, Vite, TailwindCSS, Material-UI
- Backend: .NET 6, Entity Framework Core, MySQL
- Deployment: Docker, AWS App Runner, ECR
```

#### Step 2.5: Update .gitignore

Create root-level `.gitignore`:

```gitignore
# Root level ignores
.DS_Store
.env
.env.local
.env.*.local
*.log
.vscode/
.idea/

# Build artifacts
dist/
build/
bin/
obj/

# Dependencies
node_modules/
packages/

# Docker
.docker-volumes/

# App-specific (defer to sub-directories)
apps/*/bin/
apps/*/obj/
apps/*/dist/
apps/*/node_modules/
```

Commit changes:
```bash
git add .
git commit -m "Setup: Update configuration for monorepo structure"
```

---

### Phase 3: Docker Production Configuration

#### Step 3.1: Create Multi-Stage Dockerfile

Create `docker/Dockerfile`:

```dockerfile
# =============================================================================
# Stage 1: Build React SPA with Vite
# =============================================================================
FROM node:20-alpine AS web-build

WORKDIR /app/web

# Copy package files
COPY apps/web/package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY apps/web/ ./

# Build production bundle
RUN npm run build

# =============================================================================
# Stage 2: Build .NET API
# =============================================================================
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS api-build

WORKDIR /app/api

# Copy solution and project files for restore
COPY apps/api/*.sln ./
COPY apps/api/Togetherhood.API/*.csproj ./Togetherhood.API/
COPY apps/api/Togetherhood.Application/*.csproj ./Togetherhood.Application/
COPY apps/api/Togetherhood.Data/*.csproj ./Togetherhood.Data/
COPY apps/api/Togetherhood.Libraries/*.csproj ./Togetherhood.Libraries/

# Restore dependencies
RUN dotnet restore

# Copy remaining source code
COPY apps/api/ ./

# Build and publish
RUN dotnet publish Togetherhood.API/Togetherhood.API.csproj \
    -c Release \
    -o /app/publish \
    --no-restore

# =============================================================================
# Stage 3: Runtime - Combine API + SPA
# =============================================================================
FROM mcr.microsoft.com/dotnet/aspnet:6.0

WORKDIR /app

# Install curl for health checks
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# Copy published API
COPY --from=api-build /app/publish ./

# Copy built SPA to wwwroot (served by .NET)
COPY --from=web-build /app/web/dist ./wwwroot

# Create non-root user for security
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# Expose port (App Runner uses 8080 by default)
EXPOSE 8080

# Configure Kestrel to listen on port 8080
ENV ASPNETCORE_URLS=http://+:8080
ENV ASPNETCORE_ENVIRONMENT=Production

# Health check endpoint
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8080/api/health || exit 1

# Start the application
ENTRYPOINT ["dotnet", "Togetherhood.API.dll"]
```

#### Step 3.2: Create .dockerignore

Create `docker/.dockerignore`:

```dockerignore
# Version control
.git/
.gitignore
.gitattributes

# Documentation
*.md
docs/

# CI/CD
.github/
.gitlab-ci.yml

# Development files
.vscode/
.idea/
.vs/
*.suo
*.user
*.DotSettings

# Build artifacts
**/bin/
**/obj/
**/dist/
**/build/
**/out/
**/node_modules/

# Logs
*.log
logs/

# Environment files
.env*
!.env.example

# OS files
.DS_Store
Thumbs.db

# Docker
docker-compose*.yml
Dockerfile*
.dockerignore
```

#### Step 3.3: Update .NET Program.cs for Static File Serving

Edit `apps/api/Togetherhood.API/Program.cs` to add static file serving:

```csharp
// ... existing code ...

var app = builder.Build();

// Configure middleware pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

// IMPORTANT: Serve static files from wwwroot
app.UseDefaultFiles();
app.UseStaticFiles();

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();

// Map API controllers
app.MapControllers();

// SPA fallback - serve index.html for any non-API routes
app.MapFallbackToFile("index.html");

app.Run();
```

#### Step 3.4: Add Health Check Endpoint

Create a simple health check controller at `apps/api/Togetherhood.API/Controllers/HealthController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;

namespace Togetherhood.API.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class HealthController : ControllerBase
    {
        [HttpGet]
        public IActionResult Get()
        {
            return Ok(new
            {
                status = "healthy",
                timestamp = DateTime.UtcNow,
                version = "1.0.0"
            });
        }
    }
}
```

#### Step 3.5: Test Docker Build

```bash
# Build the image (from root of monorepo)
docker build -f docker/Dockerfile -t togetherhood-app:local .

# Run the container
docker run -d -p 8080:8080 --name togetherhood-test togetherhood-app:local

# Test health endpoint
curl http://localhost:8080/api/health

# Test SPA
curl http://localhost:8080/

# Check logs
docker logs togetherhood-test

# Stop and remove
docker stop togetherhood-test
docker rm togetherhood-test
```

Commit changes:
```bash
git add .
git commit -m "Docker: Add multi-stage Dockerfile for production deployment"
```

---

### Phase 4: Local Development Setup

#### Step 4.1: Create docker-compose.yml

Create `docker/docker-compose.yml`:

```yaml
version: '3.9'

services:
  # MySQL Database
  db:
    image: mysql:8.0
    container_name: togetherhood-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: togetherhood_api
      MYSQL_USER: togetherhood
      MYSQL_PASSWORD: togetherhood123
    ports:
      - "3306:3306"
    volumes:
      - db-data:/var/lib/mysql
      - ./init-db:/docker-entrypoint-initdb.d
    networks:
      - togetherhood-network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-prootpassword"]
      interval: 10s
      timeout: 5s
      retries: 5

  # .NET API
  api:
    build:
      context: ../apps/api
      dockerfile: ../../docker/Dockerfile.api.dev
    container_name: togetherhood-api
    restart: unless-stopped
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=http://+:5000
      - ConnectionStrings__DefaultConnection=Server=db;Database=togetherhood_api;User=togetherhood;Password=togetherhood123;
    ports:
      - "5000:5000"
    volumes:
      - ../apps/api:/app
      - /app/bin
      - /app/obj
    depends_on:
      db:
        condition: service_healthy
    networks:
      - togetherhood-network
    command: dotnet watch run --project Togetherhood.API/Togetherhood.API.csproj --no-launch-profile

  # React + Vite Dev Server
  web:
    build:
      context: ../apps/web
      dockerfile: ../../docker/Dockerfile.web.dev
    container_name: togetherhood-web
    restart: unless-stopped
    environment:
      - VITE_API_URL=http://api:5000/api
      - VITE_ENV=development
      - VITE_PORT=3000
    ports:
      - "3000:3000"
    volumes:
      - ../apps/web:/app
      - /app/node_modules
    depends_on:
      - api
    networks:
      - togetherhood-network
    command: npm run dev

volumes:
  db-data:
    driver: local

networks:
  togetherhood-network:
    driver: bridge
```

#### Step 4.2: Create Development Dockerfiles

Create `docker/Dockerfile.api.dev`:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:6.0

WORKDIR /app

# Install dotnet-ef for migrations
RUN dotnet tool install --global dotnet-ef
ENV PATH="${PATH}:/root/.dotnet/tools"

# Copy solution and restore
COPY *.sln ./
COPY */*.csproj ./
RUN for file in $(ls *.csproj); do mkdir -p ${file%.*}/ && mv $file ${file%.*}/; done
RUN dotnet restore

# Copy remaining source
COPY . ./

EXPOSE 5000

CMD ["dotnet", "watch", "run", "--project", "Togetherhood.API/Togetherhood.API.csproj", "--no-launch-profile"]
```

Create `docker/Dockerfile.web.dev`:

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy source
COPY . ./

EXPOSE 3000

CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0"]
```

#### Step 4.3: Update Web Environment Configuration

Create `apps/web/.env.docker`:

```env
VITE_API_URL=http://localhost:5000/api
VITE_ENV=development
VITE_PORT=3000
```

#### Step 4.4: Create Local Setup Script

Create `scripts/local-setup.sh`:

```bash
#!/bin/bash

set -e

echo "🚀 Togetherhood Local Development Setup"
echo "======================================="

# Check prerequisites
command -v docker >/dev/null 2>&1 || { echo "❌ Docker is not installed"; exit 1; }
command -v docker-compose >/dev/null 2>&1 || { echo "❌ Docker Compose is not installed"; exit 1; }

echo "✅ Prerequisites satisfied"

# Navigate to docker directory
cd "$(dirname "$0")/../docker"

echo "🏗️  Building containers..."
docker-compose build

echo "🚀 Starting services..."
docker-compose up -d

echo "⏳ Waiting for services to be ready..."
sleep 10

echo "🔍 Checking service health..."
docker-compose ps

echo ""
echo "✅ Local environment is ready!"
echo ""
echo "📋 Access URLs:"
echo "   - Web App: http://localhost:3000"
echo "   - API: http://localhost:5000"
echo "   - API Health: http://localhost:5000/api/health"
echo "   - Database: localhost:3306"
echo ""
echo "🛠️  Useful commands:"
echo "   - View logs: docker-compose logs -f"
echo "   - Stop services: docker-compose down"
echo "   - Restart service: docker-compose restart <service-name>"
echo ""
```

Make it executable:
```bash
chmod +x scripts/local-setup.sh
```

#### Step 4.5: Test Local Development

```bash
# Run setup script
./scripts/local-setup.sh

# Or manually
cd docker
docker-compose up --build

# Test endpoints
curl http://localhost:5000/api/health
curl http://localhost:3000

# View logs
docker-compose logs -f api
docker-compose logs -f web
docker-compose logs -f db

# Stop
docker-compose down
```

Commit changes:
```bash
git add .
git commit -m "Docker: Add local development environment with docker-compose"
```

---

### Phase 5: AWS Infrastructure Setup

#### Step 5.1: Create ECR Repository

```bash
# Set variables
AWS_REGION="us-east-1"
ECR_REPO_NAME="togetherhood-app"

# Create ECR repository
aws ecr create-repository \
    --repository-name ${ECR_REPO_NAME} \
    --region ${AWS_REGION} \
    --image-scanning-configuration scanOnPush=true \
    --encryption-configuration encryptionType=AES256

# Get repository URI (save this for later)
ECR_REPO_URI=$(aws ecr describe-repositories \
    --repository-names ${ECR_REPO_NAME} \
    --region ${AWS_REGION} \
    --query 'repositories[0].repositoryUri' \
    --output text)

echo "ECR Repository URI: ${ECR_REPO_URI}"
```

**Save the ECR_REPO_URI** - you'll need it for GitHub Actions configuration.

#### Step 5.2: Create IAM Role for App Runner

Create a file `aws-config/app-runner-role-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "build.apprunner.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Create the role:

```bash
# Create IAM role
aws iam create-role \
    --role-name TogetherhoodAppRunnerECRAccessRole \
    --assume-role-policy-document file://aws-config/app-runner-role-policy.json

# Attach ECR read policy
aws iam attach-role-policy \
    --role-name TogetherhoodAppRunnerECRAccessRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSAppRunnerServicePolicyForECRAccess

# Get role ARN (save this)
ROLE_ARN=$(aws iam get-role \
    --role-name TogetherhoodAppRunnerECRAccessRole \
    --query 'Role.Arn' \
    --output text)

echo "Role ARN: ${ROLE_ARN}"
```

#### Step 5.3: Create App Runner Service Configuration

Create a file `aws-config/apprunner.json`:

```json
{
  "ServiceName": "togetherhood-app-production",
  "SourceConfiguration": {
    "ImageRepository": {
      "ImageIdentifier": "<ECR_REPO_URI>:prod",
      "ImageRepositoryType": "ECR",
      "ImageConfiguration": {
        "Port": "8080",
        "RuntimeEnvironmentVariables": {
          "ASPNETCORE_ENVIRONMENT": "Production"
        }
      }
    },
    "AuthenticationConfiguration": {
      "AccessRoleArn": "<ROLE_ARN>"
    },
    "AutoDeploymentsEnabled": true
  },
  "InstanceConfiguration": {
    "Cpu": "1024",
    "Memory": "2048"
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
```

**Note:** Replace `<ECR_REPO_URI>` and `<ROLE_ARN>` with actual values from previous steps.

Create the service:

```bash
# Replace placeholders
sed -i "s|<ECR_REPO_URI>|${ECR_REPO_URI}|g" aws-config/apprunner.json
sed -i "s|<ROLE_ARN>|${ROLE_ARN}|g" aws-config/apprunner.json

# Create App Runner service
aws apprunner create-service \
    --cli-input-json file://aws-config/apprunner.json \
    --region ${AWS_REGION}

# Get service URL (save this)
SERVICE_URL=$(aws apprunner list-services \
    --region ${AWS_REGION} \
    --query "ServiceSummaryList[?ServiceName=='togetherhood-app-production'].ServiceUrl" \
    --output text)

echo "App Runner Service URL: https://${SERVICE_URL}"
```

#### Step 5.4: Configure Environment Variables (Optional)

If you need to add environment variables (API keys, database connection strings, etc.):

```bash
# Option 1: Use AWS Secrets Manager
aws secretsmanager create-secret \
    --name togetherhood-app/production/config \
    --description "Production configuration for Togetherhood App" \
    --secret-string '{
      "ConnectionStrings__DefaultConnection": "Server=...;Database=...;User=...;Password=...",
      "JwtSettings__Secret": "your-secret-key",
      "SendGrid__ApiKey": "your-sendgrid-key"
    }' \
    --region ${AWS_REGION}

# Option 2: Add directly to App Runner service
aws apprunner update-service \
    --service-arn <SERVICE_ARN> \
    --source-configuration ImageRepository={ImageConfiguration={RuntimeEnvironmentVariables={KEY1=VALUE1,KEY2=VALUE2}}} \
    --region ${AWS_REGION}
```

**Best Practice:** Use AWS Secrets Manager and reference them in your application code.

#### Step 5.5: Setup Auto-Scaling (Optional)

```bash
aws apprunner update-service \
    --service-arn <SERVICE_ARN> \
    --auto-scaling-configuration-arn arn:aws:apprunner:${AWS_REGION}:<ACCOUNT_ID>:autoscalingconfiguration/DefaultConfiguration/1/00000000000000000000000000000001 \
    --region ${AWS_REGION}
```

---

### Phase 6: CI/CD Pipeline Configuration

#### Step 6.1: Create GitHub Actions Workflow

Create `.github/workflows/cd-production.yml`:

```yaml
name: Deploy to Production

on:
  push:
    branches:
      - main
  workflow_dispatch:

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: togetherhood-app
  IMAGE_TAG: ${{ github.sha }}

jobs:
  build-and-deploy:
    name: Build and Deploy to App Runner
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
          # Build and push with commit SHA tag
          docker build \
            -f docker/Dockerfile \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:prod \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:latest \
            .

          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:prod
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest

      - name: Trigger App Runner Deployment
        run: |
          echo "Image pushed to ECR with 'prod' tag"
          echo "App Runner will automatically detect and deploy the new image"
          echo "Image: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:prod"

      - name: Wait for deployment to complete
        run: |
          # App Runner auto-deploys when 'prod' tag is updated
          # Wait 60 seconds before checking
          sleep 60

          # Check service status
          aws apprunner list-operations \
            --service-arn <SERVICE_ARN> \
            --region ${{ env.AWS_REGION }} \
            --max-results 1

      - name: Deployment Summary
        run: |
          echo "### Deployment Summary 🚀" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "- **Environment:** Production" >> $GITHUB_STEP_SUMMARY
          echo "- **Image Tag:** ${{ env.IMAGE_TAG }}" >> $GITHUB_STEP_SUMMARY
          echo "- **ECR Repository:** ${{ env.ECR_REPOSITORY }}" >> $GITHUB_STEP_SUMMARY
          echo "- **Region:** ${{ env.AWS_REGION }}" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "App Runner will automatically deploy the new image." >> $GITHUB_STEP_SUMMARY
```

**Note:** Replace `<SERVICE_ARN>` with your actual App Runner service ARN.

#### Step 6.2: Create PR Validation Workflow

Create `.github/workflows/ci-pr.yml`:

```yaml
name: PR Validation

on:
  pull_request:
    branches:
      - main
      - develop

jobs:
  validate:
    name: Validate Build
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '6.0.x'

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Restore .NET dependencies
        working-directory: apps/api
        run: dotnet restore

      - name: Build .NET
        working-directory: apps/api
        run: dotnet build --configuration Release --no-restore

      - name: Run .NET tests
        working-directory: apps/api
        run: dotnet test --no-build --configuration Release --verbosity normal

      - name: Install Node dependencies
        working-directory: apps/web
        run: npm ci

      - name: Build React app
        working-directory: apps/web
        run: npm run build

      - name: Lint React app
        working-directory: apps/web
        run: npm run lint

      - name: Test Docker build
        run: |
          docker build -f docker/Dockerfile -t togetherhood-app:pr-${{ github.event.pull_request.number }} .

      - name: Validation Summary
        run: |
          echo "### Validation Results ✅" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "- ✅ .NET build successful" >> $GITHUB_STEP_SUMMARY
          echo "- ✅ .NET tests passed" >> $GITHUB_STEP_SUMMARY
          echo "- ✅ React build successful" >> $GITHUB_STEP_SUMMARY
          echo "- ✅ React linting passed" >> $GITHUB_STEP_SUMMARY
          echo "- ✅ Docker image builds successfully" >> $GITHUB_STEP_SUMMARY
```

#### Step 6.3: Configure GitHub Secrets

Add the following secrets to your GitHub repository:

```bash
# Navigate to: GitHub Repository → Settings → Secrets and variables → Actions

# Add these secrets:
AWS_ACCESS_KEY_ID=<your-access-key-id>
AWS_SECRET_ACCESS_KEY=<your-secret-access-key>
APP_RUNNER_SERVICE_ARN=<your-service-arn>
```

Or using GitHub CLI:

```bash
gh secret set AWS_ACCESS_KEY_ID --body "<your-access-key-id>"
gh secret set AWS_SECRET_ACCESS_KEY --body "<your-secret-access-key>"
gh secret set APP_RUNNER_SERVICE_ARN --body "<your-service-arn>"
```

#### Step 6.4: Setup Branch Protection

```bash
# Using GitHub CLI
gh api repos/Togetherhood/TogetherhoodApp/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"contexts":["Validate Build"]}' \
  --field enforce_admins=true \
  --field required_pull_request_reviews='{"required_approving_review_count":1}' \
  --field restrictions=null
```

Or configure via GitHub UI:
1. Go to **Settings** → **Branches**
2. Add branch protection rule for `main`
3. Enable:
   - ✅ Require pull request reviews before merging
   - ✅ Require status checks to pass before merging
   - ✅ Require branches to be up to date before merging

Commit workflows:
```bash
git add .github/
git commit -m "CI/CD: Add GitHub Actions workflows for deployment"
git push origin main
```

---

### Phase 7: Testing & Validation

#### Step 7.1: Local Development Testing

**Test Checklist:**

```bash
# 1. Start local environment
cd docker
docker-compose up --build

# 2. Verify all containers are running
docker-compose ps
# Expected: db, api, web all in "Up" state

# 3. Test database connectivity
docker exec -it togetherhood-db mysql -u togetherhood -ptogetherhood123 -e "SHOW DATABASES;"

# 4. Test API health endpoint
curl http://localhost:5000/api/health
# Expected: {"status":"healthy",...}

# 5. Test API functionality (example)
curl http://localhost:5000/api/partners
# Expected: JSON response or 401 if auth required

# 6. Test web app
curl http://localhost:3000
# Expected: HTML response with React app

# 7. Test hot reload - make a change to a React component
# Expected: Browser auto-refreshes with changes

# 8. Test API hot reload - make a change to a controller
# Expected: API restarts automatically

# 9. Check logs for errors
docker-compose logs api
docker-compose logs web
docker-compose logs db

# 10. Stop environment
docker-compose down
```

#### Step 7.2: Docker Production Build Testing

```bash
# 1. Build production image
docker build -f docker/Dockerfile -t togetherhood-app:test .

# 2. Run production container
docker run -d -p 8080:8080 --name togetherhood-prod-test \
  -e ASPNETCORE_ENVIRONMENT=Production \
  -e ConnectionStrings__DefaultConnection="Server=host.docker.internal;Database=togetherhood_api;User=togetherhood;Password=togetherhood123;" \
  togetherhood-app:test

# 3. Wait for startup
sleep 10

# 4. Test health endpoint
curl http://localhost:8080/api/health

# 5. Test SPA is served
curl -I http://localhost:8080/
# Expected: 200 OK with text/html content-type

# 6. Test API endpoint
curl http://localhost:8080/api/partners

# 7. Test SPA routing (should return index.html for non-API routes)
curl -I http://localhost:8080/admin/partners
# Expected: 200 OK with text/html

# 8. Check container logs
docker logs togetherhood-prod-test

# 9. Verify static files are served
curl -I http://localhost:8080/assets/
# Expected: 200 or 404 depending on actual assets

# 10. Clean up
docker stop togetherhood-prod-test
docker rm togetherhood-prod-test
```

#### Step 7.3: CI/CD Pipeline Testing

```bash
# 1. Create test branch
git checkout -b test/ci-cd-pipeline

# 2. Make a trivial change
echo "# Test" >> README.md
git add README.md
git commit -m "Test: Trigger CI pipeline"

# 3. Push branch
git push origin test/ci-cd-pipeline

# 4. Create pull request
gh pr create --title "Test: CI/CD Pipeline" --body "Testing CI/CD workflow"

# 5. Monitor GitHub Actions
# - Go to GitHub Actions tab
# - Verify "PR Validation" workflow runs
# - Check all steps pass

# 6. Merge PR to main
gh pr merge --squash

# 7. Monitor deployment workflow
# - Verify "Deploy to Production" workflow runs
# - Check Docker build succeeds
# - Check ECR push succeeds
# - Check App Runner deployment triggers

# 8. Verify deployment
sleep 120  # Wait for App Runner to deploy
curl https://<your-app-runner-url>/api/health

# 9. Clean up test branch
git branch -d test/ci-cd-pipeline
git push origin --delete test/ci-cd-pipeline
```

#### Step 7.4: Production Deployment Validation

```bash
# 1. Get App Runner service URL
SERVICE_URL=$(aws apprunner list-services \
    --region us-east-1 \
    --query "ServiceSummaryList[?ServiceName=='togetherhood-app-production'].ServiceUrl" \
    --output text)

echo "Service URL: https://${SERVICE_URL}"

# 2. Test health endpoint
curl https://${SERVICE_URL}/api/health

# 3. Test SPA loads
curl -I https://${SERVICE_URL}/

# 4. Test API endpoints (with auth if required)
curl https://${SERVICE_URL}/api/partners

# 5. Check App Runner service status
aws apprunner describe-service \
    --service-arn <SERVICE_ARN> \
    --region us-east-1 \
    --query 'Service.Status'
# Expected: "RUNNING"

# 6. Check recent operations
aws apprunner list-operations \
    --service-arn <SERVICE_ARN> \
    --region us-east-1 \
    --max-results 5

# 7. View CloudWatch logs
aws logs tail /aws/apprunner/togetherhood-app-production/application \
    --follow \
    --region us-east-1

# 8. Test from browser
# Open https://${SERVICE_URL} in browser
# - Verify SPA loads correctly
# - Verify routing works
# - Verify API calls work
# - Check browser console for errors
```

#### Step 7.5: Performance & Security Testing

```bash
# 1. Load test (basic)
ab -n 100 -c 10 https://${SERVICE_URL}/api/health

# 2. SSL/TLS check
echo | openssl s_client -connect ${SERVICE_URL}:443 -servername ${SERVICE_URL} 2>/dev/null | openssl x509 -noout -text

# 3. Security headers check
curl -I https://${SERVICE_URL}/
# Check for: X-Frame-Options, X-Content-Type-Options, etc.

# 4. Response time monitoring
time curl -s https://${SERVICE_URL}/api/health

# 5. Check for common vulnerabilities
# Use OWASP ZAP or similar tools for comprehensive security testing
```

---

### Phase 8: Production Cutover

#### Step 8.1: Pre-Deployment Checklist

- [ ] All tests passing in CI/CD
- [ ] Docker image built successfully
- [ ] App Runner service created and healthy
- [ ] Environment variables configured
- [ ] Database connection string correct
- [ ] Health check endpoint responding
- [ ] Monitoring/logging configured (CloudWatch)
- [ ] Rollback plan reviewed and understood
- [ ] Stakeholders notified of deployment window

#### Step 8.2: Deployment

```bash
# 1. Verify main branch is stable
git checkout main
git pull origin main

# 2. Tag release
git tag -a v1.0.0 -m "Initial monorepo production release"
git push origin v1.0.0

# 3. Trigger deployment (if not auto-triggered)
gh workflow run cd-production.yml

# 4. Monitor deployment
gh run list --workflow=cd-production.yml
gh run watch

# 5. Verify deployment succeeded
aws apprunner list-operations \
    --service-arn <SERVICE_ARN> \
    --region us-east-1 \
    --max-results 1
```

#### Step 8.3: Post-Deployment Validation

```bash
# 1. Smoke tests
curl https://${SERVICE_URL}/api/health
curl -I https://${SERVICE_URL}/

# 2. Functional tests
# - User login
# - Partner management
# - Course creation
# - Any critical user flows

# 3. Monitor logs for errors
aws logs tail /aws/apprunner/togetherhood-app-production/application \
    --follow \
    --region us-east-1

# 4. Check CloudWatch metrics
# - Request count
# - Response times
# - Error rates
# - CPU/Memory utilization

# 5. Verify database connectivity
# Run a query that touches the database through API
```

#### Step 8.4: Update DNS (if applicable)

If you're migrating from an existing domain:

```bash
# 1. Get App Runner custom domain settings
# Follow AWS documentation to add custom domain

# 2. Update DNS records
# Add CNAME record pointing to App Runner URL

# 3. Wait for DNS propagation
# Can take 5 minutes to 48 hours

# 4. Verify DNS resolution
nslookup yourdomain.com
```

#### Step 8.5: Deprecate Old Deployments

Once new deployment is stable (recommend waiting 24-48 hours):

```bash
# 1. Update old repositories README with deprecation notice
# 2. Disable old CI/CD pipelines
# 3. Archive old Elastic Beanstalk environments (don't delete yet)
# 4. Update team documentation with new repository links
# 5. Notify team of new deployment URLs
```

---

## AWS Infrastructure Setup

### ECR Repository Configuration

**Repository Name:** `togetherhood-app`
**Region:** `us-east-1` (or your preferred region)
**Scan on Push:** Enabled
**Encryption:** AES256
**Tag Immutability:** Disabled (to allow `prod` tag updates)

**Lifecycle Policy:**

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 10 production images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["prod"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {
        "type": "expire"
      }
    },
    {
      "rulePriority": 2,
      "description": "Keep last 5 latest images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["latest"],
        "countType": "imageCountMoreThan",
        "countNumber": 5
      },
      "action": {
        "type": "expire"
      }
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
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

Apply lifecycle policy:

```bash
aws ecr put-lifecycle-policy \
    --repository-name togetherhood-app \
    --lifecycle-policy-text file://aws-config/ecr-lifecycle-policy.json \
    --region us-east-1
```

### App Runner Service Configuration

**Service Name:** `togetherhood-app-production`
**CPU:** 1 vCPU (1024)
**Memory:** 2 GB (2048)
**Port:** 8080
**Auto Scaling:**
- Min instances: 1
- Max instances: 10
- Concurrency: 100

**Health Check:**
- Protocol: HTTP
- Path: `/api/health`
- Interval: 10 seconds
- Timeout: 5 seconds
- Healthy threshold: 1
- Unhealthy threshold: 5

**Environment Variables:**

| Variable | Value | Source |
|----------|-------|--------|
| `ASPNETCORE_ENVIRONMENT` | `Production` | Direct |
| `ConnectionStrings__DefaultConnection` | `Server=...` | Secrets Manager |
| `JwtSettings__Secret` | `<secret>` | Secrets Manager |
| `SendGrid__ApiKey` | `<api-key>` | Secrets Manager |
| `AWS__Region` | `us-east-1` | Direct |

### IAM Roles & Policies

**Role 1: ECR Access Role (for App Runner)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "build.apprunner.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Attached Policies:
- `AWSAppRunnerServicePolicyForECRAccess` (AWS managed)

**Role 2: GitHub Actions Deployment Role**

Permissions needed:
- `ecr:GetAuthorizationToken`
- `ecr:BatchCheckLayerAvailability`
- `ecr:PutImage`
- `ecr:InitiateLayerUpload`
- `ecr:UploadLayerPart`
- `ecr:CompleteLayerUpload`
- `apprunner:ListOperations`
- `apprunner:DescribeService`

---

## Testing & Validation

### Local Development Tests

| Test | Command | Expected Result |
|------|---------|----------------|
| Start environment | `docker-compose up` | All 3 containers running |
| API health | `curl http://localhost:5000/api/health` | `{"status":"healthy"}` |
| Web app loads | `curl http://localhost:3000` | HTML with React app |
| Database connection | `docker exec togetherhood-db mysql -u togetherhood -p -e "SHOW DATABASES;"` | Lists databases |
| API hot reload | Edit controller → save | API restarts automatically |
| Web hot reload | Edit React component → save | Browser auto-refreshes |

### Production Build Tests

| Test | Command | Expected Result |
|------|---------|----------------|
| Docker build | `docker build -f docker/Dockerfile .` | Image built successfully |
| Container runs | `docker run -p 8080:8080 <image>` | Container starts without errors |
| Health check | `curl http://localhost:8080/api/health` | Returns healthy status |
| SPA served | `curl http://localhost:8080/` | Returns index.html |
| API works | `curl http://localhost:8080/api/partners` | Returns API response |
| Routing works | `curl http://localhost:8080/admin/partners` | Returns index.html (SPA handles route) |

### CI/CD Tests

| Test | Trigger | Expected Result |
|------|---------|----------------|
| PR validation | Open PR to `main` | All checks pass |
| .NET build | PR workflow | Build succeeds |
| .NET tests | PR workflow | All tests pass |
| React build | PR workflow | Build succeeds, no errors |
| React lint | PR workflow | No linting errors |
| Docker build | PR workflow | Image builds successfully |
| ECR push | Merge to `main` | Image pushed with tags |
| App Runner deploy | ECR image updated | Service deploys new version |

### Production Validation Tests

| Test | Method | Expected Result |
|------|--------|----------------|
| Health endpoint | `curl https://<url>/api/health` | Returns 200 OK |
| SPA loads | Open in browser | App loads correctly |
| API endpoints | Test critical endpoints | All work as expected |
| User flows | Manual testing | Login, CRUD operations work |
| Performance | Load test | Response times acceptable |
| Logs | CloudWatch | No errors in logs |
| Monitoring | CloudWatch metrics | Metrics reporting correctly |

---

## Rollback Strategy

### Scenario 1: Bad Deployment Detected Immediately

**Symptoms:** Health checks fail, 5xx errors, application crashes

**Action:**

```bash
# 1. Identify last known good image
aws ecr describe-images \
    --repository-name togetherhood-app \
    --region us-east-1 \
    --query 'sort_by(imageDetails,& imagePushedAt)[-5:]'

# 2. Tag previous working image as 'prod'
PREVIOUS_IMAGE_DIGEST=<digest-of-working-image>
aws ecr batch-get-image \
    --repository-name togetherhood-app \
    --image-ids imageDigest=$PREVIOUS_IMAGE_DIGEST \
    --region us-east-1

aws ecr put-image \
    --repository-name togetherhood-app \
    --image-tag prod \
    --image-manifest "$(aws ecr batch-get-image --repository-name togetherhood-app --image-ids imageDigest=$PREVIOUS_IMAGE_DIGEST --query 'images[0].imageManifest' --output text)" \
    --region us-east-1

# 3. App Runner will auto-deploy the reverted image
# Monitor deployment
aws apprunner list-operations \
    --service-arn <SERVICE_ARN> \
    --region us-east-1

# Expected: Service redeploys within 2-3 minutes
```

**Time to Rollback:** 3-5 minutes

### Scenario 2: Issues Discovered After Deployment

**Symptoms:** Bug reports, data issues, performance degradation

**Action:**

```bash
# 1. Revert git commit
git revert HEAD
git push origin main

# 2. This triggers new CI/CD pipeline with previous code
# 3. Monitor GitHub Actions for new build
# 4. Verify deployment after ~10 minutes
```

**Time to Rollback:** 10-15 minutes (full CI/CD cycle)

### Scenario 3: Complete Service Failure

**Symptoms:** App Runner service unhealthy, can't recover

**Action:**

```bash
# 1. Pause auto-deployments
aws apprunner update-service \
    --service-arn <SERVICE_ARN> \
    --source-configuration '{"AutoDeploymentsEnabled":false}' \
    --region us-east-1

# 2. Manually specify working image
aws apprunner update-service \
    --service-arn <SERVICE_ARN> \
    --source-configuration ImageRepository={ImageIdentifier=<ECR_URI>:<WORKING_TAG>} \
    --region us-east-1

# 3. Re-enable auto-deployments after fix
aws apprunner update-service \
    --service-arn <SERVICE_ARN> \
    --source-configuration '{"AutoDeploymentsEnabled":true}' \
    --region us-east-1
```

**Time to Rollback:** 5-10 minutes

### Scenario 4: Database Migration Issue

**Symptoms:** Database schema incompatible with rolled-back code

**Action:**

```bash
# 1. Identify problematic migration
cd apps/api
dotnet ef migrations list

# 2. Rollback migration
dotnet ef database update <PREVIOUS_MIGRATION_NAME>

# 3. Deploy code rollback (as in Scenario 1 or 2)
```

**Time to Rollback:** 10-20 minutes (depends on migration complexity)

---

## Post-Migration Checklist

### Immediate (Day 1)

- [ ] Verify production deployment is healthy
- [ ] Monitor error logs for 24 hours
- [ ] Test critical user workflows
- [ ] Verify database connectivity
- [ ] Check CloudWatch metrics
- [ ] Notify team of new repository and URLs
- [ ] Update team documentation
- [ ] Update runbooks and incident response docs

### Week 1

- [ ] Monitor performance metrics
- [ ] Collect developer feedback on new workflow
- [ ] Verify CI/CD pipeline stability
- [ ] Check AWS costs (compare to previous)
- [ ] Review and optimize Docker image size
- [ ] Fine-tune App Runner instance configuration
- [ ] Document any issues and resolutions

### Week 2-4

- [ ] Deprecate old repositories (mark as archived)
- [ ] Remove old CI/CD pipelines
- [ ] Terminate old Elastic Beanstalk environments (after confirming stability)
- [ ] Clean up unused AWS resources
- [ ] Optimize docker-compose for developer experience
- [ ] Create additional documentation as needed
- [ ] Set up long-term monitoring and alerting

### Long-term

- [ ] Review and optimize App Runner costs
- [ ] Consider CloudFront for static asset CDN (Phase 2)
- [ ] Evaluate need for staging environment
- [ ] Plan migration to ECS/Fargate if needed (Phase 2)
- [ ] Implement advanced monitoring (distributed tracing, etc.)
- [ ] Set up automated security scanning
- [ ] Review and update disaster recovery procedures

---

## Troubleshooting Guide

### Issue: Docker build fails during React build stage

**Symptoms:**
```
ERROR: failed to solve: process "/bin/sh -c npm run build" did not complete successfully
```

**Solutions:**
1. Check for TypeScript errors: `cd apps/web && npm run build`
2. Verify all dependencies are in package.json
3. Check Node version compatibility
4. Increase Docker build memory: `docker build --memory=4g ...`

### Issue: .NET container fails to start

**Symptoms:**
```
Application startup exception: Unable to connect to database
```

**Solutions:**
1. Verify connection string is correct
2. Check database is accessible from container
3. Verify database credentials
4. Check firewall/security group rules
5. Review `docker logs <container-name>` for detailed errors

### Issue: App Runner deployment stuck in "Operation in progress"

**Symptoms:**
- Deployment doesn't complete after 10+ minutes
- Service status stuck in "OPERATION_IN_PROGRESS"

**Solutions:**
1. Check CloudWatch logs for errors
2. Verify health check endpoint is responding
3. Check ECR image pull permissions
4. Verify IAM role has correct policies
5. Contact AWS support if issue persists

### Issue: SPA routes return 404 in production

**Symptoms:**
- Direct navigation to `/admin/partners` returns 404
- Works in local development

**Solutions:**
1. Verify `app.MapFallbackToFile("index.html");` is in Program.cs
2. Check that static files middleware is configured
3. Ensure Vite build output is in `wwwroot/`
4. Verify .NET routing order (API routes before fallback)

### Issue: CORS errors in production

**Symptoms:**
```
Access to fetch at 'https://api.example.com/api/partners' from origin 'https://app.example.com' has been blocked by CORS policy
```

**Solutions:**
1. Since API and SPA are served from same origin, CORS should not be needed
2. Verify Vite is building with correct `base` URL
3. Check API calls are using relative paths (`/api/...`) not absolute URLs
4. Review network tab to see actual request URLs

### Issue: Environment variables not being read

**Symptoms:**
- Application can't find configuration values
- "Configuration key not found" errors

**Solutions:**
1. Verify environment variables are set in App Runner service
2. Check variable names match exactly (case-sensitive)
3. Use Secrets Manager for sensitive values
4. Review `appsettings.Production.json` for overrides
5. Check CloudWatch logs for configuration loading messages

---

## Additional Resources

### Documentation
- [Docker Multi-Stage Builds](https://docs.docker.com/build/building/multi-stage/)
- [AWS App Runner Developer Guide](https://docs.aws.amazon.com/apprunner/)
- [.NET 6 Static File Middleware](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/static-files)
- [Vite Build for Production](https://vitejs.dev/guide/build.html)
- [Git Subtree Merging](https://git-scm.com/book/en/v2/Git-Tools-Advanced-Merging)

### Tools
- [AWS CLI Documentation](https://docs.aws.amazon.com/cli/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Docker Compose Documentation](https://docs.docker.com/compose/)

### Support Channels
- GitHub Issues: `Togetherhood/TogetherhoodApp/issues`
- AWS Support: Use AWS Console support center
- Team Slack: `#togetherhood-dev` (internal)

---

## Appendix

### A. Git History Preservation Verification Script

```bash
#!/bin/bash
# verify-history.sh

echo "Verifying git history preservation..."

# Check API history
API_COMMITS=$(git log --oneline apps/api/ | wc -l)
echo "API commits in monorepo: $API_COMMITS"

# Check WEB history
WEB_COMMITS=$(git log --oneline apps/web/ | wc -l)
echo "WEB commits in monorepo: $WEB_COMMITS"

# Check for specific known commits (replace with actual commit messages)
echo "Checking for known API commits..."
git log --oneline apps/api/ | grep "DST: Fix addl datetime issues"

echo "Checking for known WEB commits..."
git log --oneline apps/web/ | grep "SESSIONS: Condense grid use"

echo "History verification complete!"
```

### B. Environment Variable Template

Create `.env.example` in root:

```env
# .NET API Configuration
ASPNETCORE_ENVIRONMENT=Development
ASPNETCORE_URLS=http://+:5000

# Database
ConnectionStrings__DefaultConnection=Server=localhost;Database=togetherhood_api;User=togetherhood;Password=your-password;

# JWT
JwtSettings__Secret=your-secret-key-min-32-chars
JwtSettings__Issuer=TogetherhoodAPI
JwtSettings__Audience=TogetherhoodWeb
JwtSettings__ExpirationMinutes=60

# AWS
AWS__Region=us-east-1
AWS__AccessKeyId=your-access-key
AWS__SecretAccessKey=your-secret-key
AWS__S3BucketName=your-bucket-name

# SendGrid
SendGrid__ApiKey=your-sendgrid-api-key
SendGrid__FromEmail=noreply@togetherhood.com
SendGrid__FromName=Togetherhood

# Twilio
Twilio__AccountSid=your-account-sid
Twilio__AuthToken=your-auth-token
Twilio__PhoneNumber=+1234567890

# Tipalti
Tipalti__ApiKey=your-tipalti-api-key
Tipalti__ApiUrl=https://ui2.sandbox.tipalti.com/

# Vite (Web App)
VITE_API_URL=http://localhost:5000/api
VITE_ENV=development
VITE_PORT=3000
```

### C. Useful Commands Reference

```bash
# Local Development
docker-compose -f docker/docker-compose.yml up          # Start all services
docker-compose -f docker/docker-compose.yml down        # Stop all services
docker-compose -f docker/docker-compose.yml logs -f api # View API logs
docker-compose -f docker/docker-compose.yml restart web # Restart web service

# Production Build
docker build -f docker/Dockerfile -t togetherhood-app . # Build image
docker run -p 8080:8080 togetherhood-app                # Run container
docker exec -it <container> /bin/bash                   # Shell into container

# AWS CLI
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <ECR_URI>
aws apprunner list-services --region us-east-1
aws apprunner describe-service --service-arn <ARN> --region us-east-1
aws logs tail /aws/apprunner/<service>/application --follow --region us-east-1

# GitHub CLI
gh repo view
gh pr create
gh pr merge
gh workflow list
gh run list
gh run watch

# Git
git log --oneline apps/api/                             # View API history
git log --oneline apps/web/                             # View WEB history
git log --graph --oneline --all                         # View all history
```

---

## Summary

This migration plan consolidates TogetherhoodWEB and TogetherhoodAPI into a unified monorepo called TogetherhoodApp, following the "Project Beowulf" vision for simplified deployment.

**Key Achievements:**
✅ Single repository with preserved git history
✅ Single Docker image deployment
✅ One-command local development (`docker-compose up`)
✅ Simplified CI/CD to AWS App Runner
✅ Reduced operational complexity
✅ Better developer experience

**Next Steps:**
1. Follow Phase 1-8 in sequence
2. Test thoroughly at each phase
3. Deploy to production with confidence
4. Monitor for 1-2 weeks before deprecating old systems
5. Document lessons learned
6. Plan for Phase 2 enhancements (CloudFront, ECS, advanced monitoring)

**Questions or Issues?**
- Create an issue in GitHub: `Togetherhood/TogetherhoodApp/issues`
- Refer to troubleshooting section above
- Consult with DevOps team

---

**Document Version:** 1.0
**Last Updated:** October 24, 2025
**Prepared By:** Claude Code
**Status:** Ready for Implementation
