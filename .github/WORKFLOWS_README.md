# GitHub Actions CI/CD Pipeline Templates

Universal reusable GitHub Actions workflows for CI/CD pipeline (Build, Deploy, Notify) supporting multiple projects and environments.

## 📋 Overview

This template provides a modular CI/CD pipeline with the following stages:

1. **Prepare** - Extract build information and determine environment (production/staging)
2. **Build** - Build Docker image and push to Amazon ECR
3. **Update Vault** - Update vault secrets via helper service
4. **Deploy** - Deploy using ArgoCD or Spinnaker
5. **Notify Success** - Send success notification to Slack
6. **Notify Failure** - Send failure notification to Slack
7. **Clean Up** - Delete non-production tags after successful deployment

## 🎯 Features

- ✅ **Universal** - Works with BACKEND, FRONTEND, DATA, QA, SRE projects
- ✅ **Modular** - Split into separate reusable workflows
- ✅ **Environment Detection** - Automatic staging/production based on tag pattern
- ✅ **Multi-tag Support** - `v*.*.* | production-* | staging-* | sre-*`
- ✅ **ArgoCD & Spinnaker** - Fallback deployment strategy
- ✅ **Vault Integration** - Automatic secret management
- ✅ **Slack Notifications** - Success and failure alerts
- ✅ **Auto Cleanup** - Delete staging tags after deployment

## 📁 Workflow Files

### Core Workflows (Reusable)

| File | Purpose | Type |
|------|---------|------|
| `prepare.yml` | Extract and validate deployment info | Reusable |
| `build.yml` | Build Docker image and push to ECR | Reusable |
| `update-vault.yml` | Update vault secrets | Reusable |
| `deploy.yml` | Deploy to ArgoCD or Spinnaker | Reusable |
| `notify-success.yml` | Send success notification | Reusable |
| `notify-failure.yml` | Send failure notification | Reusable |
| `clean-up-tag.yml` | Delete staging tags | Reusable |
| `cicd-pipeline.yml` | Main orchestrator | Template |

## 🚀 Setup Instructions

### 1. Create `.github/workflows/cicd-pipeline.yml` in Your Repository

Copy the content from the orchestrator template file to your repository:

```bash
mkdir -p .github/workflows
cp cicd-pipeline.yml .github/workflows/cicd-pipeline.yml
```

Or create it manually:

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

jobs:
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

  # ... (see full template for all jobs)
```

### 2. Configure GitHub Secrets

Add the following secrets to your repository (**Settings > Secrets and variables > Actions**):

#### AWS Credentials
```
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_ACCOUNT_ID
AWS_DEFAULT_REGION
```

#### ArgoCD Credentials
```
ARGOCD_USERNAME
ARGOCD_PASSWORD
```

#### Vault & Helper Service
```
VAULT_UPDATE_TOKEN       # Token for vault operations
```

#### Slack Integration
```
SLACK_BOT_TOKEN          # Bot token for user.lookupByEmail API
SLACK_WEBHOOK_URL        # Webhook URL for posting messages
```

## 🏷️ Tag Patterns

### Environment Detection

| Tag Pattern | Environment | Action |
|-------------|-------------|--------|
| `v1.0.0` | production | Deploy & Keep tag |
| `production-*` | production | Deploy & Keep tag |
| `staging-*` | staging | Deploy & Delete tag after success |
| `sre-*` | staging | Deploy & Delete tag after success |

**Regex Pattern Used:**
```
Production: v[0-9]+\.[0-9]+\.[0-9]+$ OR production-*
Staging:    staging-* OR sre-*
```

## 📤 Usage Examples

### Example 1: Release to Production

```bash
# Create semantic version tag
git tag v1.2.3
git push origin v1.2.3

# Pipeline will:
# 1. Detect environment = production
# 2. Build Docker image: imagor-service:v1.2.3
# 3. Update vault secrets
# 4. Deploy to ArgoCD (imagor-service-production)
# 5. Send success notification to #be-deployment-production
# 6. Keep tag (no cleanup)
```

### Example 2: Deploy to Staging

```bash
# Create staging tag
git tag staging-qa-2024-04-13
git push origin staging-qa-2024-04-13

# Pipeline will:
# 1. Detect environment = staging
# 2. Build Docker image: imagor-service:staging-qa-2024-04-13
# 3. Update vault secrets
# 4. Deploy to ArgoCD (imagor-service-staging)
# 5. Send success notification to #deployment-v4
# 6. Delete tag after successful deployment
```

### Example 3: Manual Deployment

```bash
# Trigger via GitHub UI
# Settings > Actions > Workflow Dispatch
# Input: v1.2.3

# Pipeline will:
# 1. Use provided version tag: v1.2.3
# 2. Proceed as production deployment
```

## 🔧 Customization

### Update Pipeline Variables

Edit `.github/workflows/cicd-pipeline.yml`:

```yaml
env:
  PROJECT: BACKEND  # Change to FRONTEND, DATA, etc
  CHANNEL_SLACK_STAGING: '#your-staging-channel'
  CHANNEL_SLACK_PRODUCTION: '#your-prod-channel'
```

### Update Service Name Logic

Modify `prepare.yml` to customize service name extraction:

```yaml
- name: Extract service name from repository
  id: service
  run: |
    # Current: repository name + "-service" suffix
    SERVICE_NAME="${{ github.repository }}"
    SERVICE_NAME="${SERVICE_NAME##*/}"
    SERVICE_NAME="${SERVICE_NAME}-service"
    
    # Or use custom mapping
    echo "service_name=${SERVICE_NAME}" >> $GITHUB_OUTPUT
```

### Update ArgoCD Configuration

Edit `deploy.yml` to customize ArgoCD settings:

```yaml
# Change ArgoCD URL
argocd login YOUR_ARGOCD_URL --grpc-web \
  --username ${{ secrets.ARGOCD_USERNAME }} \
  --password ${{ secrets.ARGOCD_PASSWORD }} \
  --insecure

# Update application naming convention
SERVICE_NAME_ARGOCD="${{ inputs.service_name }}-custom-${ENVIRONMENT}"
```

### Update Slack Channels

Customize notification channels in orchestrator:

```yaml
notify-success:
  with:
    channel_staging: '#your-staging-alerts'
    channel_production: '#your-prod-alerts'
```

## 📊 Reusable Workflow Outputs

### `prepare.yml` Outputs

```yaml
outputs:
  build_tag: Build tag extracted from git ref
  environment: 'production' or 'staging'
  service_name: Extracted service name
  should_deploy: 'true' or 'false'
```

## 🔐 Security Best Practices

1. **Use Branch Protection Rules** - Require reviews before merging to main
2. **Rotate Secrets Regularly** - Especially VAULT_UPDATE_TOKEN
3. **Limit Runner Access** - Use self-hosted runners on private infrastructure
4. **Audit Deployments** - Review Slack notifications and history API logs
5. **Use OIDC** - Consider GitHub OIDC for AWS credentials instead of static keys

## 📝 Troubleshooting

### Common Issues

**Issue: ArgoCD CLI download fails**
```
Solution: Run with sudo to fix permission issues
$(sudo mkdir -p /usr/local/bin && sudo curl ... && sudo chmod +x)
```

**Issue: Vault update fails**
```
Solution: Check VAULT_UPDATE_TOKEN secret and helper service connectivity
$(curl http://helper.kumpar.com/ping)
```

**Issue: Slack notification not sent**
```
Solution: Verify SLACK_BOT_TOKEN and SLACK_WEBHOOK_URL are correct
$(curl -H "Authorization: Bearer $SLACK_BOT_TOKEN" https://slack.com/api/auth.test)
```

## 📚 References

- [GitHub Actions Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [ArgoCon CD Documentation](https://argo-cd.readthedocs.io/)
- [Slack API Documentation](https://api.slack.com/)

## 📧 Support

For issues or questions about the CI/CD templates, contact the SRE team.

---

**Last Updated:** April 2026  
**Maintainer:** SRE Team  
**Status:** Production Ready ✅
