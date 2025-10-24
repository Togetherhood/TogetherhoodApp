# CI/CD Testing Strategy

## Overview

This guide covers the complete testing strategy for the Togetherhood monorepo, including unit tests, integration tests, and E2E tests in the CI/CD pipeline.

---

## Test Types

| Test Type | Tool | Scope | Runs When | Duration |
|-----------|------|-------|-----------|----------|
| **Unit Tests (API)** | xUnit | Business logic, utilities | PR + Local | < 2 min |
| **Unit Tests (Web)** | Vitest | React components, utilities | PR + Local | < 1 min |
| **Integration Tests (API)** | xUnit | API endpoints, database | PR + Local | < 5 min |
| **E2E Tests** | WebDriverIO + Cucumber | User flows | Manual/Nightly | 10-30 min |
| **Docker Build Test** | Docker | Image builds successfully | PR | 3-5 min |
| **Linting** | ESLint + dotnet format | Code quality | PR + Local | < 1 min |

---

## CI/CD Test Flow

### Pull Request Workflow

```
Developer creates PR
    ↓
GitHub Actions: ci-pr.yml
    ↓
┌─────────────────────────────────────────┐
│ Parallel Jobs:                          │
├─────────────────────────────────────────┤
│ 1. API Tests                            │
│    - dotnet restore                     │
│    - dotnet build                       │
│    - dotnet test (all test projects)    │
│    - dotnet format --verify-no-changes  │
│                                          │
│ 2. Web Tests                            │
│    - npm ci                             │
│    - npm run build                      │
│    - npm run test (vitest)              │
│    - npm run lint                       │
│                                          │
│ 3. Docker Build                         │
│    - docker build (multi-stage)         │
│    - docker run (smoke test)            │
└─────────────────────────────────────────┘
    ↓ (All pass)
PR Ready for Review
    ↓ (Approved + merged)
Deploy to Production
```

### Production Deployment Workflow

```
Merge to main
    ↓
GitHub Actions: cd-production.yml
    ↓
┌─────────────────────────────────────────┐
│ Build & Deploy                          │
├─────────────────────────────────────────┤
│ 1. Build Docker image                   │
│ 2. Run smoke tests in container         │
│ 3. Push to AWS ECR                      │
│ 4. Tag as 'prod'                        │
│ 5. App Runner auto-deploys              │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│ Post-Deployment Validation              │
├─────────────────────────────────────────┤
│ 1. Health check endpoint                │
│ 2. Smoke tests (critical endpoints)     │
│ 3. (Optional) Trigger E2E test suite    │
└─────────────────────────────────────────┘
```

---

## Unit Testing: .NET API (xUnit)

### Test Structure

```
apps/api/
├── Togetherhood.Application.Tests/
│   ├── Commands/
│   │   └── Partners/
│   │       └── CreatePartnerCommandTests.cs
│   ├── Queries/
│   │   └── Partners/
│   │       └── GetPartnerQueryTests.cs
│   └── ...
├── Togetherhood.Libraries.Tests/
│   ├── Services/
│   │   ├── SendGridServiceTests.cs
│   │   └── TwilioServiceTests.cs
│   └── ...
└── Togetherhood.API.Tests/
    ├── Controllers/
    │   └── PartnersControllerTests.cs
    └── ...
```

### Running Tests Locally

```bash
# Run all tests
cd apps/api
dotnet test

# Run specific test project
dotnet test Togetherhood.Application.Tests

# Run tests with coverage
dotnet test --collect:"XPlat Code Coverage"

# Run tests matching filter
dotnet test --filter "FullyQualifiedName~Partner"

# Run tests in parallel
dotnet test --parallel
```

### Example Test

```csharp
using Xunit;
using NSubstitute;
using Togetherhood.Application.Commands.Partners;

namespace Togetherhood.Application.Tests.Commands.Partners
{
    public class CreatePartnerCommandTests
    {
        private readonly IPartnerRepository _partnerRepository;
        private readonly CreatePartnerCommandHandler _handler;

        public CreatePartnerCommandTests()
        {
            _partnerRepository = Substitute.For<IPartnerRepository>();
            _handler = new CreatePartnerCommandHandler(_partnerRepository);
        }

        [Fact]
        public async Task Handle_ValidCommand_CreatesPartner()
        {
            // Arrange
            var command = new CreatePartnerCommand
            {
                Name = "Test Partner",
                Email = "test@example.com"
            };

            // Act
            var result = await _handler.Handle(command, CancellationToken.None);

            // Assert
            Assert.NotNull(result);
            await _partnerRepository.Received(1).AddAsync(Arg.Any<Partner>());
        }
    }
}
```

### GitHub Actions Configuration

```yaml
# .github/workflows/ci-pr.yml (partial)
api-tests:
  name: API Tests
  runs-on: ubuntu-latest

  services:
    mysql:
      image: mysql:8.0
      env:
        MYSQL_ROOT_PASSWORD: root
        MYSQL_DATABASE: togetherhood_test
      options: >-
        --health-cmd="mysqladmin ping"
        --health-interval=10s
        --health-timeout=5s
        --health-retries=5

  steps:
    - uses: actions/checkout@v4

    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: '6.0.x'

    - name: Restore dependencies
      working-directory: apps/api
      run: dotnet restore

    - name: Build
      working-directory: apps/api
      run: dotnet build --configuration Release --no-restore

    - name: Run tests
      working-directory: apps/api
      run: dotnet test --no-build --configuration Release --verbosity normal --logger "trx" --collect:"XPlat Code Coverage"

    - name: Upload test results
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: api-test-results
        path: apps/api/**/TestResults/**/*.trx

    - name: Upload coverage
      uses: codecov/codecov-action@v4
      with:
        files: apps/api/**/TestResults/**/coverage.cobertura.xml
        flags: api
```

---

## Unit Testing: React Web (Vitest)

### Test Structure

```
apps/web/
├── src/
│   ├── components/
│   │   ├── Button/
│   │   │   ├── Button.tsx
│   │   │   └── Button.test.tsx
│   │   └── ...
│   ├── hooks/
│   │   ├── usePartners.ts
│   │   └── usePartners.test.ts
│   └── helpers/
│       ├── formatDate.ts
│       └── formatDate.test.ts
└── vitest.config.ts
```

### Vitest Configuration

**`apps/web/vitest.config.ts`:**

```typescript
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: './src/test/setup.ts',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html', 'lcov'],
      exclude: [
        'node_modules/',
        'src/test/',
        '**/*.d.ts',
        '**/*.config.*',
        '**/mockData/*',
        'src/main.tsx',
      ],
    },
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
});
```

**`apps/web/src/test/setup.ts`:**

```typescript
import { expect, afterEach } from 'vitest';
import { cleanup } from '@testing-library/react';
import matchers from '@testing-library/jest-dom/matchers';

// Extend Vitest's expect with jest-dom matchers
expect.extend(matchers);

// Cleanup after each test
afterEach(() => {
  cleanup();
});
```

### Running Tests Locally

```bash
# Run all tests
cd apps/web
npm run test

# Watch mode
npm run test:watch

# With coverage
npm run test:coverage

# Run specific test file
npm run test -- Button.test.tsx

# Update snapshots
npm run test -- -u
```

### Package.json Scripts

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:ui": "vitest --ui",
    "test:coverage": "vitest run --coverage"
  },
  "devDependencies": {
    "vitest": "^1.0.0",
    "@vitest/ui": "^1.0.0",
    "@testing-library/react": "^14.0.0",
    "@testing-library/jest-dom": "^6.0.0",
    "@testing-library/user-event": "^14.0.0",
    "@vitest/coverage-v8": "^1.0.0"
  }
}
```

### Example Tests

**Component Test:**

```typescript
// src/components/Button/Button.test.tsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen, fireEvent } from '@testing-library/react';
import { Button } from './Button';

describe('Button', () => {
  it('renders with text', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByText('Click me')).toBeInTheDocument();
  });

  it('calls onClick when clicked', () => {
    const handleClick = vi.fn();
    render(<Button onClick={handleClick}>Click me</Button>);

    fireEvent.click(screen.getByText('Click me'));
    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it('is disabled when disabled prop is true', () => {
    render(<Button disabled>Click me</Button>);
    expect(screen.getByText('Click me')).toBeDisabled();
  });
});
```

**Hook Test:**

```typescript
// src/hooks/usePartners.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { renderHook, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { usePartners } from './usePartners';

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });
  return ({ children }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
};

describe('usePartners', () => {
  it('fetches partners successfully', async () => {
    const { result } = renderHook(() => usePartners(), {
      wrapper: createWrapper(),
    });

    await waitFor(() => expect(result.current.isSuccess).toBe(true));
    expect(result.current.data).toBeDefined();
  });
});
```

**Utility Test:**

```typescript
// src/helpers/formatDate.test.ts
import { describe, it, expect } from 'vitest';
import { formatDate } from './formatDate';

describe('formatDate', () => {
  it('formats date correctly', () => {
    const date = new Date('2025-10-24T12:00:00Z');
    expect(formatDate(date, 'MM/dd/yyyy')).toBe('10/24/2025');
  });

  it('handles invalid date', () => {
    expect(formatDate(null, 'MM/dd/yyyy')).toBe('');
  });
});
```

### GitHub Actions Configuration

```yaml
# .github/workflows/ci-pr.yml (partial)
web-tests:
  name: Web Tests
  runs-on: ubuntu-latest

  steps:
    - uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
        cache-dependency-path: apps/web/package-lock.json

    - name: Install dependencies
      working-directory: apps/web
      run: npm ci

    - name: Run tests
      working-directory: apps/web
      run: npm run test:coverage

    - name: Upload coverage
      uses: codecov/codecov-action@v4
      with:
        files: apps/web/coverage/lcov.info
        flags: web

    - name: Build
      working-directory: apps/web
      run: npm run build

    - name: Lint
      working-directory: apps/web
      run: npm run lint
```

---

## Integration Testing

### API Integration Tests

Integration tests for API endpoints that hit the database:

```csharp
// Togetherhood.API.Tests/Integration/PartnersControllerIntegrationTests.cs
using Microsoft.AspNetCore.Mvc.Testing;
using Xunit;

public class PartnersControllerIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public PartnersControllerIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task GetPartners_ReturnsSuccess()
    {
        // Act
        var response = await _client.GetAsync("/api/partners");

        // Assert
        response.EnsureSuccessStatusCode();
        var content = await response.Content.ReadAsStringAsync();
        Assert.NotEmpty(content);
    }
}
```

These run against a test database (MySQL service in GitHub Actions).

---

## E2E Testing

See **[E2E Testing Integration Guide](e2e-testing.md)** for detailed information on WebDriverIO + Cucumber tests.

Quick summary:
- Located in `apps/e2e/`
- Run manually or on schedule (nightly)
- Test critical user flows end-to-end
- Can be triggered after production deployment

---

## Complete CI/CD Workflow Files

### PR Validation Workflow

**`.github/workflows/ci-pr.yml`:**

```yaml
name: PR Validation

on:
  pull_request:
    branches:
      - main
      - develop

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  api-tests:
    name: API Tests
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: togetherhood_test
          MYSQL_USER: test_user
          MYSQL_PASSWORD: test_pass
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '6.0.x'

      - name: Restore dependencies
        working-directory: apps/api
        run: dotnet restore

      - name: Build
        working-directory: apps/api
        run: dotnet build --configuration Release --no-restore

      - name: Run tests
        working-directory: apps/api
        env:
          ConnectionStrings__DefaultConnection: "Server=localhost;Database=togetherhood_test;User=test_user;Password=test_pass;"
        run: |
          dotnet test \
            --no-build \
            --configuration Release \
            --verbosity normal \
            --logger "trx;LogFileName=test-results.trx" \
            --collect:"XPlat Code Coverage" \
            -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=opencover

      - name: Check code formatting
        working-directory: apps/api
        run: dotnet format --verify-no-changes --verbosity diagnostic

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: api-test-results
          path: apps/api/**/TestResults/**/*.trx

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: apps/api/**/TestResults/**/coverage.opencover.xml
          flags: api
          name: api-coverage

  web-tests:
    name: Web Tests
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: apps/web/package-lock.json

      - name: Install dependencies
        working-directory: apps/web
        run: npm ci

      - name: Run tests
        working-directory: apps/web
        run: npm run test:coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: apps/web/coverage/lcov.info
          flags: web
          name: web-coverage

      - name: Build
        working-directory: apps/web
        run: npm run build

      - name: Lint
        working-directory: apps/web
        run: npm run lint

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: web-dist
          path: apps/web/dist/

  docker-build:
    name: Docker Build Test
    runs-on: ubuntu-latest
    needs: [api-tests, web-tests]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build Docker image
        run: |
          docker build \
            -f docker/Dockerfile \
            -t togetherhood-app:pr-${{ github.event.pull_request.number }} \
            .

      - name: Run smoke tests
        run: |
          # Start container
          docker run -d \
            --name togetherhood-test \
            -p 8080:8080 \
            -e ASPNETCORE_ENVIRONMENT=Production \
            togetherhood-app:pr-${{ github.event.pull_request.number }}

          # Wait for startup
          sleep 15

          # Test health endpoint
          curl -f http://localhost:8080/api/health || exit 1

          # Test SPA loads
          curl -f -I http://localhost:8080/ || exit 1

          # Cleanup
          docker stop togetherhood-test
          docker rm togetherhood-test

  pr-summary:
    name: PR Summary
    runs-on: ubuntu-latest
    needs: [api-tests, web-tests, docker-build]
    if: always()

    steps:
      - name: Generate summary
        run: |
          echo "### PR Validation Results 🎯" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "| Check | Status |" >> $GITHUB_STEP_SUMMARY
          echo "|-------|--------|" >> $GITHUB_STEP_SUMMARY
          echo "| API Tests | ${{ needs.api-tests.result == 'success' && '✅ Passed' || '❌ Failed' }} |" >> $GITHUB_STEP_SUMMARY
          echo "| Web Tests | ${{ needs.web-tests.result == 'success' && '✅ Passed' || '❌ Failed' }} |" >> $GITHUB_STEP_SUMMARY
          echo "| Docker Build | ${{ needs.docker-build.result == 'success' && '✅ Passed' || '❌ Failed' }} |" >> $GITHUB_STEP_SUMMARY
```

### Production Deployment Workflow

**`.github/workflows/cd-production.yml`:**

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
    name: Build and Deploy
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
          docker build \
            -f docker/Dockerfile \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:prod \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:latest \
            --cache-from type=registry,ref=$ECR_REGISTRY/$ECR_REPOSITORY:latest \
            --cache-to type=inline \
            .

          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:prod
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest

      - name: Wait for App Runner deployment
        run: |
          echo "Waiting for App Runner to detect and deploy new image..."
          sleep 120

      - name: Verify deployment
        env:
          SERVICE_URL: ${{ secrets.APP_RUNNER_SERVICE_URL }}
        run: |
          # Health check
          curl -f https://${SERVICE_URL}/api/health || exit 1

          # Verify version (if you add version endpoint)
          # curl https://${SERVICE_URL}/api/version | jq

      - name: Deployment summary
        run: |
          echo "### Deployment Summary 🚀" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "- **Environment:** Production" >> $GITHUB_STEP_SUMMARY
          echo "- **Image Tag:** \`${{ env.IMAGE_TAG }}\`" >> $GITHUB_STEP_SUMMARY
          echo "- **ECR Repository:** ${{ env.ECR_REPOSITORY }}" >> $GITHUB_STEP_SUMMARY
          echo "- **Deployed At:** $(date -u +'%Y-%m-%d %H:%M:%S UTC')" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "App Runner will automatically deploy the new image." >> $GITHUB_STEP_SUMMARY
```

---

## Test Coverage Goals

### Targets

| Component | Coverage Goal |
|-----------|--------------|
| API Business Logic | > 80% |
| API Controllers | > 70% |
| Web Components | > 75% |
| Web Utilities | > 90% |
| Overall | > 75% |

### Monitoring Coverage

View coverage reports in:
- **Local**: `apps/api/TestResults/` and `apps/web/coverage/`
- **GitHub**: Codecov integration shows coverage on PRs
- **CI/CD**: Uploaded as artifacts in GitHub Actions

---

## Best Practices

### ✅ DO

- ✅ Write tests before fixing bugs (TDD)
- ✅ Keep tests fast (mock external dependencies)
- ✅ Test edge cases and error handling
- ✅ Use descriptive test names
- ✅ Follow AAA pattern (Arrange, Act, Assert)
- ✅ Run tests locally before pushing
- ✅ Fix failing tests immediately
- ✅ Review test coverage on PRs

### ❌ DON'T

- ❌ Skip tests to make CI pass
- ❌ Write tests that depend on other tests
- ❌ Use real external services in unit tests
- ❌ Commit code with failing tests
- ❌ Ignore test warnings
- ❌ Test implementation details
- ❌ Write overly complex test setups

---

## Troubleshooting

### Tests fail locally but pass in CI

- Check environment variables
- Verify database connection strings
- Check Node/npm versions match CI
- Clear build artifacts: `dotnet clean` / `npm run clean`

### Tests are slow

- Mock external HTTP calls
- Use in-memory database for unit tests
- Run tests in parallel
- Reduce test data size

### Flaky tests

- Avoid timing dependencies
- Mock date/time in tests
- Use proper async/await patterns
- Check for race conditions

---

## Next Steps

- ✅ Set up vitest in web app
- ✅ Ensure all test projects build
- ✅ Configure GitHub Actions with test workflows
- ✅ Set up Codecov integration
- ✅ Train team on testing best practices

**Related Guides:**
- [E2E Testing Integration](e2e-testing.md)
- [AWS Infrastructure Setup](aws-infrastructure-setup.md)
- [Staging Environment Setup](staging-environment.md)
