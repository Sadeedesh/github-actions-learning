name: Node.js Build and Test

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - run: npm install
      
      - run: npm run build
      
      - run: npm test

- uses: actions/checkout@v3

- uses: actions/checkout@v3
  with:
    fetch-depth: 0           # Full history
    ref: main               # Specific branch
    token: ${{ secrets.GITHUB_TOKEN }}

    - uses: actions/setup-node@v3
  with:
    node-version: '18'

- uses: actions/setup-python@v4
  with:
    python-version: '3.10'

- uses: actions/setup-dotnet@v3
  with:
    dotnet-version: '6.0'

    - run: npm install
- run: npm ci              # Cleaner install for CI
- run: yarn install

- run: npm run build

{
  "scripts": {
    "build": "webpack --mode production",
    "test": "jest",
    "lint": "eslint src/"
  }
}

- run: npm test
- run: npm run test:coverage

- uses: actions/setup-node@v3
  with:
    node-version: '18'
    cache: 'npm'      # Caches node_modules

- uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [14, 16, 18, 19]
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      
      - run: npm ci
      - run: npm test

- name: Build
  run: npm run build

- name: Upload build artifact
  uses: actions/upload-artifact@v3
  with:
    name: build-output
    path: dist/

- name: Upload coverage
  uses: codecov/codecov-action@v3
  with:
    files: ./coverage/coverage-final.json
    flags: unittests
    fail_ci_if_error: true

- name: Publish test results
  uses: EnricoMi/publish-unit-test-result-action@v2
  if: always()
  with:
    files: test-results/*.xml

name: Build, Test, and Report

on:
  push:
    branches: [main, develop]
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
        uses: actions/checkout@v3
      
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
        uses: actions/upload-artifact@v3
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

- name: Run ESLint
  run: npx eslint src/ --format json --output-file eslint-report.json
  continue-on-error: true

- name: Check formatting
  run: npx prettier --check src/

- name: SonarQube Scan
  uses: sonarsource/sonarqube-scan-action@master
  env:
    SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

- name: Build
  run: npm run build
  if: github.ref == 'refs/heads/main'

- name: Deploy
  run: npm run deploy
  if: success()

- name: Cleanup on failure
  run: npm run cleanup
  if: failure()


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


- name: Run quick checks
  run: npm run lint
  if: github.event_name == 'pull_request'

- name: Run full test suite
  run: npm test
  if: github.ref == 'refs/heads/main'

- run: npm test -- --verbose- run: npm test -- --verbose

- run: npm test -- src/components/__tests__


