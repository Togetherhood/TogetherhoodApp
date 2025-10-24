# E2E Testing Integration Guide

## Overview

This guide covers integrating the existing **TogetherhoodAutomation** E2E test suite (WebDriverIO + Cucumber) into the monorepo.

---

## Current State

**Repository**: `TogetherhoodAutomation` (separate repo)
**Framework**: WebDriverIO + Cucumber (BDD)
**Language**: TypeScript
**Reports**: Allure

### Existing Structure

```
TogetherhoodAutomation/
├── src/
│   ├── features/          # Cucumber feature files (Gherkin)
│   ├── step-definitions/  # Step implementations
│   ├── pages/             # Page Object Model
│   └── support/           # Utilities, helpers
├── data/                  # Test data
├── package.json
├── wdio.conf.ts          # WebDriverIO configuration
└── tsconfig.json
```

---

## Decision: Include E2E Tests in Monorepo?

### Option 1: Include in Monorepo (Recommended)

**Location**: `apps/e2e/`

**Pros:**
- ✅ Single repository for all code
- ✅ Version control aligned with app code
- ✅ Easier to run tests against local dev environment
- ✅ Simpler CI/CD integration
- ✅ Better discoverability for developers

**Cons:**
- ❌ Larger monorepo
- ❌ E2E tests slow down CI if not managed properly

**Recommendation**: ✅ **Include in monorepo** but run E2E tests separately (nightly, manual, or post-deployment)

### Option 2: Keep Separate Repository

**Pros:**
- ✅ Smaller monorepo
- ✅ E2E tests don't affect app CI/CD speed
- ✅ Can evolve independently

**Cons:**
- ❌ Need to keep test suite in sync with app changes
- ❌ Developers might not run E2E tests locally
- ❌ Extra repository to manage

---

## Migration Plan (Option 1)

### Step 1: Add E2E to Monorepo Structure

```bash
cd /path/to/TogetherhoodApp

# Create e2e directory
mkdir -p apps/e2e

# Copy existing automation code
cp -r /path/to/TogetherhoodAutomation/* apps/e2e/

# Remove git directory
rm -rf apps/e2e/.git

# Commit to monorepo
git add apps/e2e/
git commit -m "feat: Add E2E test suite to monorepo"
```

### Step 2: Update E2E Configuration

**`apps/e2e/wdio.conf.ts`** - Update baseUrl to use environment variable:

```typescript
import type { Options } from '@wdio/types';

export const config: Options.Testrunner = {
    runner: 'local',
    specs: [
        './src/features/**/*.feature'
    ],
    exclude: [],

    maxInstances: 5,

    capabilities: [{
        browserName: 'chrome',
        'goog:chromeOptions': {
            args: ['--headless', '--disable-gpu', '--no-sandbox']
        }
    }],

    logLevel: 'info',
    bail: 0,
    baseUrl: process.env.E2E_BASE_URL || 'http://localhost:3000',

    waitforTimeout: 10000,
    connectionRetryTimeout: 120000,
    connectionRetryCount: 3,

    framework: 'cucumber',
    reporters: [
        'spec',
        ['allure', {
            outputDir: 'allure-results',
            disableWebdriverStepsReporting: true,
            disableWebdriverScreenshotsReporting: false,
        }]
    ],

    cucumberOpts: {
        require: ['./src/step-definitions/**/*.ts'],
        backtrace: false,
        requireModule: ['tsconfig-paths/register'],
        dryRun: false,
        failFast: false,
        snippets: true,
        source: true,
        strict: false,
        tagExpression: '',
        timeout: 60000,
        ignoreUndefinedDefinitions: false
    },

    // Hooks
    beforeSession: function (config, capabilities, specs) {
        require('ts-node').register({ transpileOnly: true });
    }
};
```

### Step 3: Update Package.json Scripts

**`apps/e2e/package.json`** - Add convenient scripts:

```json
{
  "name": "@togetherhood/e2e",
  "version": "1.0.0",
  "description": "E2E tests for Togetherhood App",
  "scripts": {
    "test": "wdio run wdio.conf.ts",
    "test:local": "cross-env E2E_BASE_URL=http://localhost:3000 wdio run wdio.conf.ts",
    "test:dev": "cross-env E2E_BASE_URL=https://dev.togetherhood.com wdio run wdio.conf.ts",
    "test:staging": "cross-env E2E_BASE_URL=https://staging.togetherhood.com wdio run wdio.conf.ts",
    "test:prod": "cross-env E2E_BASE_URL=https://app.togetherhood.com wdio run wdio.conf.ts",
    "test:headless": "cross-env HEADLESS=true wdio run wdio.conf.ts",
    "report": "allure generate ./allure-results --clean -o ./allure-report && allure open ./allure-report",
    "clean": "rm -rf allure-results allure-report"
  },
  "devDependencies": {
    "@cucumber/cucumber": "11.3.0",
    "@wdio/cli": "^9.20.0",
    "@wdio/cucumber-framework": "^9.20.0",
    "@wdio/local-runner": "^9.20.0",
    "@wdio/spec-reporter": "^9.20.0",
    "@wdio/allure-reporter": "^9.20.0",
    "allure-commandline": "^2.29.0",
    "cross-env": "^7.0.3",
    "ts-node": "^10.9.2",
    "typescript": "^5.4.5",
    "webdriverio": "^9.20.0"
  }
}
```

### Step 4: Add Environment Configuration

**`apps/e2e/.env.example`:**

```env
# Base URL for tests
E2E_BASE_URL=http://localhost:3000

# Test credentials (use test accounts only!)
E2E_ADMIN_EMAIL=admin@example.com
E2E_ADMIN_PASSWORD=test_password

E2E_INSTRUCTOR_EMAIL=instructor@example.com
E2E_INSTRUCTOR_PASSWORD=test_password

E2E_PARTNER_EMAIL=partner@example.com
E2E_PARTNER_PASSWORD=test_password

# Browser settings
HEADLESS=true
BROWSER_NAME=chrome

# Timeouts
E2E_TIMEOUT=30000
```

**Add to `.gitignore`:**
```
apps/e2e/.env
apps/e2e/.env.local
apps/e2e/allure-results/
apps/e2e/allure-report/
apps/e2e/node_modules/
```

---

## Updated Monorepo Structure

```
TogetherhoodApp/
├── apps/
│   ├── api/                    # .NET 6 Web API
│   ├── web/                    # React + Vite SPA
│   └── e2e/                    # E2E Tests (NEW)
│       ├── src/
│       │   ├── features/       # Cucumber features (.feature files)
│       │   ├── step-definitions/
│       │   ├── pages/          # Page Object Model
│       │   └── support/        # Utilities
│       ├── data/               # Test data
│       ├── allure-results/     # Test results (ignored)
│       ├── allure-report/      # Generated reports (ignored)
│       ├── package.json
│       ├── wdio.conf.ts
│       ├── tsconfig.json
│       └── .env.example
├── docker/
├── docs/
└── ...
```

---

## Running E2E Tests

### Locally (Against Local Development Environment)

```bash
# 1. Start local dev environment
docker-compose -f docker/docker-compose.yml up

# 2. In another terminal, run E2E tests
cd apps/e2e
npm install
npm run test:local

# 3. View report
npm run report
```

### Against Deployed Environments

```bash
# Test against dev environment
cd apps/e2e
npm run test:dev

# Test against staging
npm run test:staging

# Test against production (use with caution!)
npm run test:prod
```

### In Docker (Isolated Environment)

Create **`apps/e2e/Dockerfile`:**

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci

# Install Chrome/Chromium for WebDriverIO
RUN apk add --no-cache \
    chromium \
    nss \
    freetype \
    harfbuzz \
    ca-certificates \
    ttf-freefont

# Set Chrome path
ENV CHROME_BIN=/usr/bin/chromium-browser
ENV PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true

# Copy test code
COPY . .

# Run tests
CMD ["npm", "run", "test:headless"]
```

Run in Docker:

```bash
docker build -t togetherhood-e2e -f apps/e2e/Dockerfile apps/e2e
docker run --rm \
  -e E2E_BASE_URL=http://host.docker.internal:3000 \
  togetherhood-e2e
```

---

## CI/CD Integration

### Option A: Scheduled Nightly Runs (Recommended)

**`.github/workflows/e2e-nightly.yml`:**

```yaml
name: E2E Tests (Nightly)

on:
  schedule:
    # Run at 2 AM UTC every night
    - cron: '0 2 * * *'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to test'
        required: true
        default: 'staging'
        type: choice
        options:
          - local
          - dev
          - staging
          - production

jobs:
  e2e-tests:
    name: E2E Tests
    runs-on: ubuntu-latest

    strategy:
      fail-fast: false
      matrix:
        browser: [chrome, firefox]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: apps/e2e/package-lock.json

      - name: Install dependencies
        working-directory: apps/e2e
        run: npm ci

      - name: Run E2E tests
        working-directory: apps/e2e
        env:
          E2E_BASE_URL: ${{ secrets.STAGING_URL }}
          E2E_ADMIN_EMAIL: ${{ secrets.E2E_ADMIN_EMAIL }}
          E2E_ADMIN_PASSWORD: ${{ secrets.E2E_ADMIN_PASSWORD }}
          BROWSER_NAME: ${{ matrix.browser }}
        run: npm run test:headless

      - name: Upload Allure results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: allure-results-${{ matrix.browser }}
          path: apps/e2e/allure-results/

      - name: Generate Allure report
        if: always()
        working-directory: apps/e2e
        run: npm run report

      - name: Upload Allure report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: allure-report-${{ matrix.browser }}
          path: apps/e2e/allure-report/

      - name: Notify on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
          payload: |
            {
              "text": "E2E Tests Failed!",
              "attachments": [{
                "color": "danger",
                "fields": [
                  {"title": "Browser", "value": "${{ matrix.browser }}", "short": true},
                  {"title": "Workflow", "value": "${{ github.workflow }}", "short": true},
                  {"title": "Run", "value": "${{ github.run_number }}", "short": true}
                ]
              }]
            }
```

### Option B: Manual Trigger After Deployment

**`.github/workflows/cd-production.yml`** - Add E2E trigger at end:

```yaml
# ... existing deployment steps ...

- name: Trigger E2E tests (optional)
  if: success()
  uses: actions/github-script@v7
  with:
    script: |
      await github.rest.actions.createWorkflowDispatch({
        owner: context.repo.owner,
        repo: context.repo.repo,
        workflow_id: 'e2e-nightly.yml',
        ref: 'main',
        inputs: {
          environment: 'production'
        }
      });
```

### Option C: Run on Every PR (Not Recommended - Too Slow)

Only do this if E2E tests are very fast (< 5 minutes):

```yaml
# .github/workflows/ci-pr.yml
e2e-tests:
  name: E2E Tests
  runs-on: ubuntu-latest
  needs: [docker-build]

  steps:
    # ... setup steps ...
    # Start docker-compose
    # Wait for services
    # Run E2E tests
```

---

## Test Data Management

### Test Database

E2E tests should use a dedicated test database with known seed data:

```bash
# In docker-compose, add test database service
db-test:
  image: mysql:8.0
  environment:
    MYSQL_ROOT_PASSWORD: rootpassword
    MYSQL_DATABASE: togetherhood_e2e
  volumes:
    - ./apps/e2e/data/seed.sql:/docker-entrypoint-initdb.d/seed.sql
```

**`apps/e2e/data/seed.sql`:**

```sql
-- Create test users
INSERT INTO users (email, password_hash, role) VALUES
('admin@example.com', 'hashed_password', 'Admin'),
('instructor@example.com', 'hashed_password', 'Instructor'),
('partner@example.com', 'hashed_password', 'Partner');

-- Create test partners
INSERT INTO partners (name, email, status) VALUES
('Test Partner', 'partner@example.com', 'Active');

-- Create test courses
INSERT INTO courses (name, partner_id, status) VALUES
('Test Course', 1, 'Active');
```

### Test Isolation

Each E2E test should:
1. Set up its own test data (or use known seed data)
2. Clean up after itself
3. Not depend on other tests

Example in step definition:

```typescript
// src/step-definitions/partners.steps.ts
import { Given, When, Then } from '@cucumber/cucumber';

Given('a test partner exists', async function () {
  // Create test partner via API
  this.testPartner = await createTestPartner({
    name: 'Test Partner',
    email: `test-${Date.now()}@example.com`
  });
});

After(async function () {
  // Cleanup test data
  if (this.testPartner) {
    await deleteTestPartner(this.testPartner.id);
  }
});
```

---

## Page Object Model

Maintain Page Object Model for maintainability:

```typescript
// src/pages/partners.page.ts
class PartnersPage {
  get addButton() { return $('button[data-testid="add-partner"]'); }
  get nameInput() { return $('input[name="name"]'); }
  get emailInput() { return $('input[name="email"]'); }
  get saveButton() { return $('button[type="submit"]'); }

  async open() {
    await browser.url('/admin/partners');
  }

  async createPartner(name: string, email: string) {
    await this.addButton.click();
    await this.nameInput.setValue(name);
    await this.emailInput.setValue(email);
    await this.saveButton.click();
  }
}

export default new PartnersPage();
```

Use in step definitions:

```typescript
// src/step-definitions/partners.steps.ts
import PartnersPage from '../pages/partners.page';

When('I create a new partner with name {string} and email {string}',
  async (name: string, email: string) => {
    await PartnersPage.open();
    await PartnersPage.createPartner(name, email);
  }
);
```

---

## Best Practices

### ✅ DO

- ✅ Run E2E tests separately from unit/integration tests
- ✅ Use test-specific data (don't depend on production data)
- ✅ Clean up test data after each test
- ✅ Use Page Object Model for maintainability
- ✅ Add data-testid attributes to make selectors stable
- ✅ Run E2E tests on schedule (nightly) or manually
- ✅ Keep E2E tests focused on critical user journeys
- ✅ Use meaningful scenario names in Gherkin

### ❌ DON'T

- ❌ Run E2E tests on every PR (too slow)
- ❌ Test every edge case in E2E (use unit tests)
- ❌ Use brittle CSS selectors (use data-testid)
- ❌ Create dependencies between tests
- ❌ Run E2E tests against production (use staging)
- ❌ Hardcode test data (use fixtures or factories)
- ❌ Skip tests that fail (fix them!)

---

## Reporting

### Allure Reports

Allure provides rich test reports with screenshots, logs, and history:

**View locally:**
```bash
cd apps/e2e
npm run report
```

**Publish to GitHub Pages (Optional):**

```yaml
# .github/workflows/e2e-nightly.yml
- name: Publish Allure report
  if: always()
  uses: peaceiris/actions-gh-pages@v3
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    publish_dir: apps/e2e/allure-report
    destination_dir: e2e-reports/${{ github.run_number }}
```

Access reports at: `https://togetherhood.github.io/TogetherhoodApp/e2e-reports/<run-number>`

---

## Troubleshooting

### WebDriverIO can't connect to browser

**Solution**: Install Chrome/Chromium:
```bash
# macOS
brew install --cask google-chrome

# Ubuntu
wget -q -O - https://dl-ssl.google.com/linux/linux_signing_key.pub | sudo apt-key add -
sudo sh -c 'echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google.list'
sudo apt-get update
sudo apt-get install google-chrome-stable
```

### Tests timeout

**Solution**: Increase timeout in `wdio.conf.ts`:
```typescript
cucumberOpts: {
  timeout: 120000  // 2 minutes
}
```

### Element not found

**Solution**: Use explicit waits:
```typescript
await browser.waitUntil(async () => {
  return await element.isDisplayed();
}, {
  timeout: 10000,
  timeoutMsg: 'Element was not displayed after 10s'
});
```

---

## Summary

**Decision**: ✅ Include E2E tests in monorepo at `apps/e2e/`

**Benefits:**
- Single source of truth
- Tests version-controlled with application
- Easier local development
- Simplified CI/CD integration

**CI/CD Strategy:**
- Run on schedule (nightly)
- Optionally trigger after production deployment
- Don't run on every PR (too slow)

**Next Steps:**
1. Copy TogetherhoodAutomation to `apps/e2e/`
2. Update configuration for monorepo
3. Set up GitHub Actions for nightly runs
4. Create test data seed scripts
5. Document critical user journeys to test

**Related Guides:**
- [CI/CD Testing Strategy](cicd-testing.md)
- [Staging Environment Setup](staging-environment.md)
- [AWS Infrastructure Setup](aws-infrastructure-setup.md)
