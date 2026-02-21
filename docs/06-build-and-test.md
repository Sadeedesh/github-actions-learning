# Build and Test

## Build Process Overview

A **build** converts source code into executable artifacts. A **test** verifies your code works correctly.

## Node.js Build & Test Example

### Basic Setup

```yaml
name: Node.js Build and Test

on:
  push:
    branches: [main, working-intermediate-sadeedesh]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
      
      - run: npm install
      
      - run: npm run build
      
      - run: npm test
```

## Step-by-Step Breakdown

### 1. Checkout Code

Always start with checkout:

```yaml
- uses: actions/checkout@v4
```

Options:

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0           # Full history
    ref: main               # Specific branch
    token: ${{ secrets.GITHUB_TOKEN }}
```

### 2. Setup Environment

Setup the right runtime:

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '18'

- uses: actions/setup-python@v4
  with:
    python-version: '3.10'

- uses: actions/setup-dotnet@v3
  with:
    dotnet-version: '6.0'
```

### 3. Install Dependencies

```yaml
- run: npm install
- run: npm ci              # Cleaner install for CI
- run: yarn install
```

### 4. Build Application

```yaml
- run: npm run build
```

Your `package.json`:

```json
{
  "scripts": {
    "build": "webpack --mode production",
    "test": "jest",
    "lint": "eslint src/"
  }
}
```

### 5. Run Tests

```yaml
- run: npm test
- run: npm run test:coverage
```

## Caching Dependencies

Speed up workflows by caching:

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '18'
    cache: 'npm'      # Caches node_modules
```

Or manual cache:

```yaml
- uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

## Matrix Testing

Test multiple versions:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [14, 16, 18, 19]
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      
      - run: npm ci
      - run: npm test
```

## Artifacts & Reporting

### Upload Artifacts

```yaml
- name: Build
  run: npm run build

- name: Upload build artifact
  uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: dist/
```

### Upload Coverage Reports

```yaml
- name: Upload coverage
  uses: codecov/codecov-action@v3
  with:
    files: ./coverage/coverage-final.json
    flags: unittests
    fail_ci_if_error: true
```

### Publish Test Results

```yaml
- name: Publish test results
  uses: EnricoMi/publish-unit-test-result-action@v2
  if: always()
  with:
    files: test-results/*.xml
```

## Real-World Complete Example

```yaml
name: Build, Test, and Report

on:
  push:
    branches: [main, working-intermediate-sadeedesh]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [16, 18]
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Lint code
        run: npm run lint
        continue-on-error: true
      
      - name: Build application
        run: npm run build
      
      - name: Run tests
        run: npm test -- --coverage
      
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results-${{ matrix.node-version }}
          path: coverage/
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/coverage-final.json
          flags: unittests
          name: codecov-umbrella
          fail_ci_if_error: false
```

## Linting and Code Quality

### ESLint

```yaml
- name: Run ESLint
  run: npx eslint src/ --format json --output-file eslint-report.json
  continue-on-error: true
```

### Prettier

```yaml
- name: Check formatting
  run: npx prettier --check src/
```

### SonarQube

```yaml
- name: SonarQube Scan
  uses: sonarsource/sonarqube-scan-action@master
  env:
    SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

## Conditional Steps

Run steps conditionally:

```yaml
- name: Build
  run: npm run build
  if: github.ref == 'refs/heads/main'

- name: Deploy
  run: npm run deploy
  if: success()

- name: Cleanup on failure
  run: npm run cleanup
  if: failure()
```

## Performance Optimization

### Parallel Jobs

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build
```

All three run in parallel!

### Early Exit

```yaml
- name: Run quick checks
  run: npm run lint
  if: github.event_name == 'pull_request'

- name: Run full test suite
  run: npm test
  if: github.ref == 'refs/heads/main'
```

## Troubleshooting

### Debug Output

```yaml
- run: npm test -- --verbose
```

### Run Specific Tests

```yaml
- run: npm test -- src/components/__tests__
```

### Skip Workflows for Certain Commits

In your commit message:

```
[skip ci] Updated README
```

## Best Practices

✅ **Test on multiple Node versions** (matrix)  
✅ **Cache dependencies** (npm, yarn)  
✅ **Run linting** before building  
✅ **Generate coverage reports**  
✅ **Upload artifacts** for inspection  
✅ **Use descriptive step names**  
✅ **Fail fast** on critical errors  

## Common Mistakes

❌ Not caching dependencies (slow)  
❌ Testing only one Node version  
❌ Not running linting  
❌ Forgetting to checkout code  
❌ Using `npm install` instead of `npm ci` in CI  

## Next Steps

- [Deploy to Azure](07-deploy-to-azure.md) - Deploy your application
- [TASKS.md](../TASKS.md) - Try hands-on exercises
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions) - Find more actions

---

**You now understand build and test workflows! Next, let's deploy to Azure.**
