name: Build, Test, and Report

on:
  push:
    branches:
      - main
      - working-intermediate-sadeedesh
  pull_request:
    branches:
      - main

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [16, 18]

    steps:
      # Checkout repository
      - name: Checkout code
        uses: actions/checkout@v4

      # Setup Node.js
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: npm

      # Install dependencies (CI optimized)
      - name: Install dependencies
        run: npm ci

      # Lint code (non-blocking)
      - name: Lint code
        run: npm run lint
        continue-on-error: true

      # Build application
      - name: Build application
        run: npm run build

      # Run tests with coverage
      - name: Run tests
        run: npm test -- --coverage

      # Upload test results artifact
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results-${{ matrix.node-version }}
          path: coverage/

      # Upload coverage report
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage/coverage-final.json
          flags: unittests
          fail_ci_if_error: false