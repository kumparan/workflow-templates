# CI/CD Universal Template - Usage Example

This file shows how to use the universal CI/CD template from another repository.

## 📁 Files Structure

Your repository should have this structure:

```
imagor/
├── .github/
│   └── workflows/
│       └── cicd-pipeline.yml          ← Copy from template or use reference
├── Dockerfile
├── src/
├── README.md
└── ...
```

## 📄 Step 1: Create `.github/workflows/cicd-pipeline.yml`

Create this file in your service repository (e.g., in imagor):

```yaml
name: CI/CD Pipeline

on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'
      - 'production-*'
      - 'staging-*'
      - 'sre-*'
  workflow_dispatch:
    inputs:
      version_tag:
        description: 'Version tag (optional, for manual versioning)'
        required: false
        type: string

env:
  PROJECT: BACKEND
  DEPLOYER: ${{ github.actor }}
  IMAGE_TAG: ${{ github.ref_name }}
  CHANNEL_SLACK_STAGING: '#deployment-v4'
  CHANNEL_SLACK_PRODUCTION: '#be-deployment-production'
  APP_VERSION: ${{ github.ref_name }}

jobs:
  # Call the universal template workflows
  prepare:
    uses: kumparan/cicd-template/.github/workflows/prepare.yml@main
    with:
      project: BACKEND

  build:
    needs: [prepare]
    if: needs.prepare.outputs.should_deploy == 'true'
    uses: kumparan/cicd-template/.github/workflows/build.yml@main
    with:
      service_name: ${{ needs.prepare.outputs.service_name }}
      build_tag: ${{ needs.prepare.outputs.build_tag }}
      environment: ${{ needs.prepare.outputs.environment }}
      should_deploy: ${{ needs.prepare.outputs.should_deploy }}
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}
      AWS_DEFAULT_REGION: ${{ secrets.AWS_DEFAULT_REGION }}

  update-vault:
    needs: [prepare, build]
    if: needs.prepare.outputs.should_deploy == 'true'
    uses: kumparan/cicd-template/.github/workflows/update-vault.yml@main
    with:
      service_name: ${{ needs.prepare.outputs.service_name }}
      build_tag: ${{ needs.prepare.outputs.build_tag }}
      environment: ${{ needs.prepare.outputs.environment }}
    secrets:
      VAULT_UPDATE_TOKEN: ${{ secrets.VAULT_UPDATE_TOKEN }}

  deploy:
    needs: [prepare, build, update-vault]
    if: needs.prepare.outputs.should_deploy == 'true'
    uses: kumparan/cicd-template/.github/workflows/deploy.yml@main
    with:
      service_name: ${{ needs.prepare.outputs.service_name }}
      build_tag: ${{ needs.prepare.outputs.build_tag }}
      environment: ${{ needs.prepare.outputs.environment }}
      project: ${{ env.PROJECT }}
    secrets:
      AWS_ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}
      AWS_DEFAULT_REGION: ${{ secrets.AWS_DEFAULT_REGION }}
      ARGOCD_USERNAME: ${{ secrets.ARGOCD_USERNAME }}
      ARGOCD_PASSWORD: ${{ secrets.ARGOCD_PASSWORD }}
      VAULT_UPDATE_TOKEN: ${{ secrets.VAULT_UPDATE_TOKEN }}

  notify-success:
    needs: [prepare, deploy]
    if: success()
    uses: kumparan/cicd-template/.github/workflows/notify-success.yml@main
    with:
      service_name: ${{ needs.prepare.outputs.service_name }}
      build_tag: ${{ needs.prepare.outputs.build_tag }}
      environment: ${{ needs.prepare.outputs.environment }}
      project: ${{ env.PROJECT }}
      channel_staging: ${{ env.CHANNEL_SLACK_STAGING }}
      channel_production: ${{ env.CHANNEL_SLACK_PRODUCTION }}
    secrets:
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

  notify-failure:
    needs: [prepare, deploy]
    if: failure()
    uses: kumparan/cicd-template/.github/workflows/notify-failure.yml@main
    with:
      service_name: ${{ needs.prepare.outputs.service_name }}
      build_tag: ${{ needs.prepare.outputs.build_tag }}
      environment: ${{ needs.prepare.outputs.environment }}
      project: ${{ env.PROJECT }}
      channel_staging: ${{ env.CHANNEL_SLACK_STAGING }}
      channel_production: ${{ env.CHANNEL_SLACK_PRODUCTION }}
    secrets:
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

  clean-up-tag:
    needs: [prepare, deploy]
    if: success()
    uses: kumparan/cicd-template/.github/workflows/clean-up-tag.yml@main
    with:
      environment: ${{ needs.prepare.outputs.environment }}
```

## 🔐 Step 2: Configure Repository Secrets

In your repository (e.g., imagor), go to:
**Settings > Secrets and variables > Actions**

Add these secrets:

| Secret | Value | Notes |
|--------|-------|-------|
| `AWS_ACCESS_KEY_ID` | `AXXX...` | From AWS IAM user |
| `AWS_SECRET_ACCESS_KEY` | `xXXX...` | From AWS IAM user |
| `AWS_ACCOUNT_ID` | `123456789` | AWS account number |
| `AWS_DEFAULT_REGION` | `us-east-1` | AWS region |
| `ARGOCD_USERNAME` | `your-user` | ArgoCD login user |
| `ARGOCD_PASSWORD` | `your-pass` | ArgoCD login password |
| `VAULT_UPDATE_TOKEN` | `0551d2d8...` | From Kresno |
| `SLACK_BOT_TOKEN` | `xoxb-xxxxx` | Bot user token |
| `SLACK_WEBHOOK_URL` | `https://hooks.slack.com/...` | Webhook URL |

## 🚀 Step 3: Deploy!

### Option A: Using Git Tags (Recommended)

```bash
# Staging deployment
git tag staging-qa-20260413
git push origin staging-qa-20260413

# Production deployment
git tag v1.0.0
git push origin v1.0.0

# Production with descriptor
git tag production-qa-final
git push origin production-qa-final
```

### Option B: Manual Trigger

1. Go to **Actions** tab in GitHub
2. Select **CI/CD Pipeline**
3. Click **Run workflow**
4. Enter version tag (e.g., v1.0.0)
5. Click **Run**

## 📊 Data Flow

```
Git Tag Push
    ↓
GitHub Workflow Triggers
    ↓
[prepare.yml]
    ↓ Outputs: build_tag, environment, service_name, should_deploy
    ├── [build.yml] - Docker build & push to ECR
    │   ↓
    ├── [update-vault.yml] - Update secrets
    │   ↓
    ├── [deploy.yml] - Deploy to ArgoCD/Spinnaker
    │   ↓
    ├── [notify-success.yml] OR [notify-failure.yml]
    │   ↓
    └── [clean-up-tag.yml] (only if staging)
        ↓
Deployment Complete
```

## 🔍 Example Execution

### Scenario: Deploy imagor-service v1.2.3 to Production

**Input:**
```bash
git tag v1.2.3
git push origin v1.2.3
```

**Processing:**

1. **prepare.yml**:
   ```
   build_tag = v1.2.3
   environment = production
   service_name = imagor-service
   should_deploy = true
   ```

2. **build.yml**:
   ```
   Docker image: imagor-service:v1.2.3
   ECR: 123456789.dkr.ecr.us-east-1.amazonaws.com/imagor-service:v1.2.3
   Status: Pushed ✓
   ```

3. **update-vault.yml**:
   ```
   Helper Service: ping http://helper.kumpar.com/ping → 200 ✓
   Update Vault: imagor-service production secrets updated ✓
   ```

4. **deploy.yml**:
   ```
   ArgoCD App: imagor-service-production
   Helm Param: deployment.image = 123456789.dkr.ecr.us-east-1.amazonaws.com/imagor-service:v1.2.3
   Sync: Application synced ✓
   Wait: Health OK + Sync OK ✓
   ```

5. **notify-success.yml**:
   ```
   Channel: #be-deployment-production
   Message:
   ✅ DEPLOYMENT SUCCEEDED: | BACKEND | production
   Service: imagor-service
   Tag: v1.2.3
   Deployed by: @your-name
   ```

6. **clean-up-tag.yml**:
   ```
   Environment is production, skip cleanup ✓
   Tag v1.2.3 remains in repository ✓
   ```

## ✅ Verification Checklist

- [ ] `.github/workflows/cicd-pipeline.yml` exists in repository
- [ ] All 9 secrets configured in GitHub
- [ ] Dockerfile exists in repository root
- [ ] Service name matches ArgoCD app name pattern
- [ ] Tag follows: `v*.*.* | production-* | staging-* | sre-*`
- [ ] ArgoCD application exists: `servicename-environment`
- [ ] Helper service accessible: `http://helper.kumpar.com/ping`
- [ ] Slack channels are correct: `#deployment-v4`, `#be-deployment-production`

## 🐛 Debug Tips

### Check Workflow Logs

1. Go to **Actions** tab
2. Find your workflow run
3. Click job name to see details
4. Check logs for errors

### Common Issues

```bash
# Check service name extraction
# Your repo name should end with service name
# e.g.: kumparan/imagor → imagor-service

# Check tag format
git tag -l | grep -E "^(v|production-|staging-|sre-)"

# Check ArgoCD app exists
argocd app get imagor-service-staging

# Check helper service
curl http://helper.kumpar.com/ping
```

## 📝 References

- [Template Repository](https://github.com/kumparan/cicd-template)
- [WORKFLOWS_README.md](.github/WORKFLOWS_README.md)
- [QUICKSTART.md](.github/QUICKSTART.md)

---

**Questions?** Ask in #sre-internal Slack channel.
