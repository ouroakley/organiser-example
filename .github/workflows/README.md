# Deployment Trigger Setup

This guide explains how to set up the deployment trigger workflow in your organiser repository. This workflow allows you to trigger builds in the main repository using GitHub Apps for secure authentication.

## Overview

The workflow:
1. Triggers on pushes to the main branch
2. Generates a JWT using the app's private key
3. Uses the JWT to get an installation token
4. Creates a deployment in the main repository
5. Triggers a build and deploy to Cloudflare Pages

## Prerequisites

1. Access to:
   - Your organiser repository
   - The main repository (`ouroakley`)
2. Organisation admin access to:
   - Create a GitHub App
   - Install the app on repositories

## Setup Steps

### 1. Create GitHub App

1. Go to your organisation's settings → GitHub Apps → New GitHub App
2. Configure the app:
   - Name: "Deployment Manager" (or similar)
   - Description: "Manages deployments from organiser repositories"
   - Homepage URL: Your organisation's website
   - Webhook: Leave disabled (not needed for deployments)
   - Permissions:
     - `Deployments`: Read & Write
     - `Contents`: Read-only
     - `Workflows`: Read-only
3. Click "Create GitHub App"
4. **IMPORTANT**: Download and save the private key (PEM file)

### 2. Install GitHub App

1. After creating the app, click "Install App"
2. Select both repositories:
   - The main repository (`ouroakley`)
   - Your organiser repository
3. Note down the installation ID:
   - Go to the main repository's settings → GitHub Apps
   - Find your app and note the installation ID
   - This ID is needed to generate installation tokens

### 3. Configure Repository Secrets

Add the following secrets to your organiser repository:

1. Go to repository → Settings → Secrets and variables → Actions
2. Add these secrets:
   - `GITHUB_APP_PRIVATE_KEY`: The private key from step 1 (PEM format)
   - `GITHUB_APP_ID`: Your app's ID (found in app settings)
   - `INSTALLATION_ID`: The installation ID from step 2
   - `TARGET_REPO`: The main repository name (e.g., `ouroakley/main`)

### 4. Add Workflow File

Add the workflow file `.github/workflows/trigger-build.yml`:

```yaml
name: Trigger Main Repository Build

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Get Installation Token
        id: get_token
        uses: actions/github-script@v7
        with:
          script: |
            const app = require('jsonwebtoken');
            const now = Math.floor(Date.now() / 1000);
            const payload = {
              iat: now,
              exp: now + 600,
              iss: process.env.GITHUB_APP_ID
            };
            const jwt = app.sign(payload, process.env.GITHUB_APP_PRIVATE_KEY, { algorithm: 'RS256' });

            const response = await github.request({
              method: 'POST',
              url: `/app/installations/${process.env.INSTALLATION_ID}/access_tokens`,
              headers: {
                authorization: `Bearer ${jwt}`,
                accept: 'application/vnd.github.v3+json'
              }
            });
            core.setOutput('token', response.data.token);
          env:
            GITHUB_APP_PRIVATE_KEY: ${{ secrets.GITHUB_APP_PRIVATE_KEY }}
            GITHUB_APP_ID: ${{ secrets.GITHUB_APP_ID }}

      - name: Create Deployment
        uses: actions/github-script@v7
        with:
          script: |
            const response = await github.request({
              method: 'POST',
              url: `/repos/${process.env.TARGET_REPO}/deployments`,
              headers: {
                authorization: `Bearer ${process.env.INSTALLATION_TOKEN}`,
                accept: 'application/vnd.github.v3+json',
                'content-type': 'application/json'
              },
              data: {
                ref: 'main',
                environment: 'production',
                description: 'Build triggered from organiser repository',
                auto_merge: true
              }
            });
            if (response.status !== 201) {
              core.setFailed(`Failed to trigger deployment: ${JSON.stringify(response.data)}`);
            }
          env:
            INSTALLATION_TOKEN: ${{ steps.get_token.outputs.token }}
            TARGET_REPO: ${{ secrets.TARGET_REPO }}
```

## How It Works

1. **JWT Generation**:
   - The workflow generates a JWT using the app's private key
   - The JWT includes:
     - Issue time (iat)
     - Expiration time (10 minutes from issue)
     - App ID as issuer
   - The JWT is signed using RS256 algorithm
   - The JWT is valid for 10 minutes

2. **Installation Token**:
   - The JWT is used to request an installation token
   - GitHub verifies the JWT and issues a token
   - The token is scoped to the installation
   - The installation token expires after 1 hour

3. **Deployment Creation**:
   - The installation token is used to create a deployment
   - GitHub verifies the token and permissions
   - The deployment triggers the main repository's workflow

## Testing

1. Push a change to your main branch
2. Check the Actions tab in your repository
3. Verify the deployment is created in the main repository
4. Monitor the build in the main repository

## Troubleshooting

Common issues and solutions:

1. **Authentication Errors**:
   - Verify the private key is correctly formatted
   - Check the installation ID is correct
   - Ensure the app is installed on both repositories

2. **Permission Issues**:
   - Verify app permissions in organisation settings
   - Check installation status in repository settings
   - Ensure the app has the required scopes

3. **Deployment Failures**:
   - Check the deployment environment exists
   - Verify the target repository name is correct
   - Review the GitHub Actions logs for details

## Security

This setup uses GitHub Apps for secure authentication. See the [GitHub App Trust Model](../../../main/.github/workflows/github-app-trust-model.md) for technical details.

## Related Documentation

- [Main Repository Setup](../../../main/.github/workflows/README.md)
- [GitHub App Trust Model](../../../main/.github/workflows/github-app-trust-model.md)
- [GitHub Apps Documentation](https://docs.github.com/en/developers/apps) 