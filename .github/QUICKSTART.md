# Quick Start Guide - Using Universal CI/CD Templates

## 📋 Quick Setup in 5 Minutes

### Step 1: Copy Template to Your Repository

```bash
# Navigate to your service repository (e.g., imagor, comment-service, etc)
cd app-repo/backend/your-service

# Create workflows directory
mkdir -p .github/workflows

# Copy the orchestrator template
curl -o .github/workflows/cicd-pipeline.yml \
  https://raw.githubusercontent.com/kumparan/cicd-template/main/.github/workflows/cicd-pipeline.yml
```

### Step 2: Configure GitHub Secrets

Go to **Settings > Secrets and variables > Actions** and add:

```
# AWS
AWS_ACCESS_KEY_ID=xxx
AWS_SECRET_ACCESS_KEY=xxx
AWS_ACCOUNT_ID=123456789
AWS_DEFAULT_REGION=us-east-1

# ArgoCD
ARGOCD_USERNAME=your-argocd-user
ARGOCD_PASSWORD=your-argocd-password

# Vault
VAULT_UPDATE_TOKEN=your-vault-token

# Slack
SLACK_BOT_TOKEN=xoxb-xxxxx
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/xxx/xxx/xxx
```

### Step 3: Customize (Optional)

Edit `.github/workflows/cicd-pipeline.yml` to change:

```yaml
env:
  PROJECT: BACKEND  # or FRONTEND, DATA
  CHANNEL_SLACK_STAGING: '#deployment-v4'
  CHANNEL_SLACK_PRODUCTION: '#be-deployment-production'
```

### Step 4: Push and Deploy!

```bash
# Create a tag and push
git tag staging-qa-2026-04-13
git push origin staging-qa-2026-04-13

# Or for production
git tag v1.0.0
git push origin v1.0.0

# Watch the workflow in GitHub Actions tab
```

## 🎯 Real-World Examples

### Example 1: Deploy imagor to Staging

```bash
cd app-repo/backend/imagor
git tag staging-imagor-qa-$(date +%Y%m%d.%H%M%S)
git push origin staging-imagor-qa-20260413.153000

# Results:
# BUILD_TAG = staging-imagor-qa-20260413.153000
# ENVIRONMENT = staging
# SERVICE_NAME = imagor-service
# Deployed to: imagor-service-staging in ArgoCD
# Channel: #deployment-v4
# Auto cleanup: Tag deleted after success ✓
```

### Example 2: Release comment-service to Production

```bash
cd app-repo/backend/comment-service
git tag v2.5.1
git push origin v2.5.1

# Results:
# BUILD_TAG = v2.5.1
# ENVIRONMENT = production
# SERVICE_NAME = comment-service
# Deployed to: comment-service-production in ArgoCD
# Channel: #be-deployment-production
# Auto cleanup: Tag kept ✓
```

### Example 3: Deploy discovery-service with production tag

```bash
cd app-repo/backend/discovery-service
git tag production-hotfix-20260413
git push origin production-hotfix-20260413

# Results:
# BUILD_TAG = production-hotfix-20260413
# ENVIRONMENT = production
# SERVICE_NAME = discovery-service
# Deployed to: discovery-service-production in ArgoCD
# Channel: #be-deployment-production
# Auto cleanup: Tag kept ✓
```

## 📊 What Happens Automatically

When you push a tag matching the patterns:

```
✅ v1.0.0                          → Production
✅ production-*                    → Production
✅ staging-*                       → Staging
✅ sre-*                          → Staging
```

The pipeline automatically:

1. **Prepares** - Extracts service name, determines environment
2. **Builds** - Creates Docker image, pushes to ECR
3. **Updates Vault** - Refreshes secrets
4. **Deploys** - Updates ArgoCD with new image
5. **Notifies** - Sends Slack message to appropriate channel
6. **Cleans** - Deletes tag if staging deployment succeeded

## 🚨 Troubleshooting

### Workflow doesn't trigger
- ✅ Check tag matches pattern: `v1.0.0`, `production-*`, `staging-*`, `sre-*`
- ✅ Verify `.github/workflows/cicd-pipeline.yml` exists
- ✅ Check GitHub Actions is enabled in Settings

### Image build fails
- ✅ Verify AWS credentials are correct
- ✅ Check Dockerfile exists in root directory
- ✅ Verify AWS_ACCOUNT_ID and AWS_DEFAULT_REGION

### Deployment fails
- ✅ Check ArgoCD username/password
- ✅ Verify ArgoCD app exists: `servicename-environment`
- ✅ Check helper service connectivity: `curl http://helper.kumpar.com/ping`

### Slack notification missing
- ✅ Verify SLACK_BOT_TOKEN is valid
- ✅ Check SLACK_WEBHOOK_URL format
- ✅ Ensure Slack channels are typed correctly (with #)

## 📖 Complete Template Reference

See [WORKFLOWS_README.md](WORKFLOWS_README.md) for:
- Detailed workflow documentation
- All configuration options
- Advanced customizations
- Security best practices

## 🎓 Learning Path

1. **Beginner** - Use defaults, push tags
2. **Intermediate** - Customize env variables, Slack channels
3. **Advanced** - Modify deployment logic, add pre/post hooks

---

**Need Help?** Contact SRE-Team in Slack #sre-internal

**Last Updated:** April 2026
