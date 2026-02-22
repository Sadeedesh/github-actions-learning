# Deploy to Azure

## Azure Deployment Overview

GitHub Actions can deploy applications directly to Azure services using built-in actions.

## Prerequisites

1. **Azure Account** - Create at [portal.azure.com](https://portal.azure.com)
2. **Service Principal** - For authentication
3. **GitHub Secrets** - Store credentials safely

## Setting Up Azure Authentication

### Create Service Principal

Using Azure CLI:

```bash
az ad sp create-for-rbac \
  --name github-actions \
  --role contributor \
  --scopes /subscriptions/{subscription-id}
```

Output will be JSON with:
- `clientId`
- `clientSecret`
- `subscriptionId`
- `tenantId`

### Store Credentials as Secret

1. Go to GitHub repository
2. Settings → Secrets and variables → Actions
3. Create `AZURE_CREDENTIALS` secret
4. Paste the JSON output

## Azure App Service Deployment

### Basic Deployment

```yaml
name: Deploy to Azure App Service

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Build application
        run: |
          npm install
          npm run build
      
      - name: Deploy to Azure App Service
        uses: azure/webapps-deploy@v2
        with:
          app-name: my-app-name
          publish-profile: ${{ secrets.AZURE_PUBLISH_PROFILE }}
          package: .
```

### Getting Publish Profile

1. Go to Azure Portal
2. App Service → your-app-name
3. Click "Get publish profile"
4. Save as `AZURE_PUBLISH_PROFILE` secret

## Complete Multi-Step Deployment

```yaml
name: Build and Deploy to Azure

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - run: npm ci
      
      - run: npm run build
      
      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-build
          path: dist/
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: app-build
          path: dist/
      
      - name: Login to Azure
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      
      - name: Deploy to App Service
        uses: azure/webapps-deploy@v2
        with:
          app-name: my-app-name
          slot-name: staging
          package: .
      
      - name: Azure logout
        run: az logout
```

## Azure CLI Commands in Workflows

### Install Azure CLI

```yaml
- name: Install Azure CLI
  uses: azure-cli/setup-azure-cli@v1
```

### Run Azure Commands

```yaml
- uses: azure/login@v1
  with:
    creds: ${{ secrets.AZURE_CREDENTIALS }}

- name: Create resource group
  run: |
    az group create \
      --name my-rg \
      --location eastus

- name: Deploy resources
  run: |
    az deployment group create \
      --resource-group my-rg \
      --template-file bicep/main.bicep \
      --parameters \
        appName=my-app \
        location=eastus
```

## Deployment Slots

Deploy to staging first, then swap:

```yaml
- name: Deploy to staging
  uses: azure/webapps-deploy@v2
  with:
    app-name: my-app-name
    slot-name: staging
    package: .

- name: Swap to production
  run: |
    az webapp deployment slot swap \
      --resource-group my-rg \
      --name my-app-name \
      --slot staging
```

## Environment-Specific Deployments

```yaml
jobs:
  deploy-staging:
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - run: |
          az login --service-principal \
            -u ${{ secrets.AZURE_CLIENT_ID }} \
            -p ${{ secrets.AZURE_CLIENT_SECRET }} \
            --tenant ${{ secrets.AZURE_TENANT_ID }}
          
          az webapp deployment source config-zip \
            --resource-group staging-rg \
            --name staging-app \
            --src-path app.zip

  deploy-production:
    needs: deploy-staging
    environment: production
    runs-on: ubuntu-latest
    steps:
      - run: |
          az login --service-principal \
            -u ${{ secrets.AZURE_CLIENT_ID }} \
            -p ${{ secrets.AZURE_CLIENT_SECRET }} \
            --tenant ${{ secrets.AZURE_TENANT_ID }}
          
          az webapp deployment source config-zip \
            --resource-group prod-rg \
            --name prod-app \
            --src-path app.zip
```

## Azure Static Web Apps

```yaml
name: Deploy to Azure Static Web Apps

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build application
        run: |
          npm install
          npm run build
      
      - name: Deploy to Static Web App
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          action: upload
          app_location: dist
          api_location: api
```

## Azure Container Instances

```yaml
- name: Build Docker image
  run: docker build . -t myregistry.azurecr.io/myapp:${{ github.sha }}

- name: Push to Azure Container Registry
  run: |
    az login --service-principal \
      -u ${{ secrets.AZURE_CLIENT_ID }} \
      -p ${{ secrets.AZURE_CLIENT_SECRET }} \
      --tenant ${{ secrets.AZURE_TENANT_ID }}
    
    docker push myregistry.azurecr.io/myapp:${{ github.sha }}
```

## Real-World Complete Example

```yaml
name: Complete Azure Deployment

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - run: npm ci
      - run: npm test
      - run: npm run build

  deploy-staging:
    if: github.ref == 'refs/heads/main'
    needs: test
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - run: npm ci && npm run build
      
      - name: Deploy to Azure
        uses: azure/webapps-deploy@v2
        with:
          app-name: my-app-staging
          publish-profile: ${{ secrets.AZURE_STAGING_PUBLISH_PROFILE }}
          package: .
      
      - name: Notify deployment
        run: echo "Deployed to staging!"

  deploy-production:
    if: github.ref == 'refs/heads/main'
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - run: npm ci && npm run build
      
      - name: Deploy to Azure
        uses: azure/webapps-deploy@v2
        with:
          app-name: my-app-production
          publish-profile: ${{ secrets.AZURE_PRODUCTION_PUBLISH_PROFILE }}
          package: .
      
      - name: Notify deployment
        run: echo "Deployed to production!"
```

## Best Practices

✅ Use **environments** for different stages  
✅ **Test before deploying** to production  
✅ Use **deployment slots** for zero downtime  
✅ Store credentials as **secrets**  
✅ **Monitor deployments** in Azure portal  
✅ Use **manual approvals** for production  

## Common Issues

### Authentication Fails

Check:
- Service Principal credentials are correct
- AZURE_CREDENTIALS secret is valid JSON
- Service Principal has required permissions

### Deployment Times Out

```yaml
timeout-minutes: 30
```

### Container Registry Login

```yaml
- name: Login to Azure Container Registry
  uses: docker/login-action@v2
  with:
    registry: myregistry.azurecr.io
    username: ${{ secrets.AZURE_REGISTRY_USERNAME }}
    password: ${{ secrets.AZURE_REGISTRY_PASSWORD }}
```

## Next Steps

- [TASKS.md](../TASKS.md) - Complete hands-on exercises
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Azure Integration Guide](https://learn.microsoft.com/en-us/azure/)

---

**Congratulations! You can now build, test, and deploy to Azure! 🎉**
