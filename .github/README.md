# GitHub Actions Reusable Workflows

This directory contains universal GitHub Actions reusable workflows for CI/CD pipeline orchestration.

## 📚 Documentation

| Document | Purpose |
|----------|---------|
| [QUICKSTART.md](QUICKSTART.md) | 5-minute setup guide |
| [WORKFLOWS_README.md](WORKFLOWS_README.md) | Complete reference documentation |
| [USAGE_EXAMPLE.md](USAGE_EXAMPLE.md) | Real-world usage examples |

## 📁 Workflows

All workflows are reusable (via `uses:` statement) and support secrets via `secrets: inherit`.

### Reusable Workflows

```
workflows/
├── prepare.yml              # Extract build info & environment
├── build.yml                # Build Docker image
├── update-vault.yml         # Update vault secrets
├── deploy.yml               # Deploy to ArgoCD/Spinnaker
├── notify-success.yml       # Send success notification
├── notify-failure.yml       # Send failure notification
├── clean-up-tag.yml         # Delete staging tags
└── cicd-pipeline.yml        # Main orchestrator (template)
```

## 🚀 Quick Start

**1. Copy orchestrator to your repository:**
```yaml
# .github/workflows/cicd-pipeline.yml
uses: kumparan/cicd-template/.github/workflows/prepare.yml@main
```

**2. Configure secrets in GitHub** (9 secrets)

**3. Push a tag:**
```bash
git tag v1.0.0
git push origin v1.0.0
```

For detailed setup, see [QUICKSTART.md](QUICKSTART.md)

## 🎯 Supported Projects

- ✅ BACKEND (comment-service, imagor, discovery-service, etc)
- ✅ FRONTEND (kumparan-mobile-app, web-text-editor, etc)
- ✅ DATA (search-service-data, mage-service, etc)
- ✅ QA (karate-graphql, remote-robo, etc)
- ✅ SRE (custom services)

## 🏷️ Tag Patterns

```
v1.0.0           → Production (kept)
production-*     → Production (kept)
staging-*        → Staging (auto-deleted)
sre-*           → Staging (auto-deleted)
```

## 🔐 Required Secrets

```
AWS_ACCESS_KEY_ID          # AWS access key
AWS_SECRET_ACCESS_KEY      # AWS secret key
AWS_ACCOUNT_ID             # AWS account number
AWS_DEFAULT_REGION         # AWS region (us-east-1)
ARGOCD_USERNAME            # ArgoCD user
ARGOCD_PASSWORD            # ArgoCD password
VAULT_UPDATE_TOKEN         # Vault API token
SLACK_BOT_TOKEN            # Slack bot token
SLACK_WEBHOOK_URL          # Slack webhook
```

## 📊 Workflow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    Git Tag Push                              │
│        (v1.0.0 | production-* | staging-* | sre-*)          │
└──────────────┬──────────────────────────────────────────────┘
               │
               ▼
         ┌──────────────┐
         │  prepare.yml │ ◄─ Extract info, determine environment
         │              │
         └──────┬───────┘
                │ outputs: build_tag, environment, service_name
                │
      ┌─────────┴────────────┐
      ▼                      ▼
  ┌──────────┐          ┌─────────────┐
  │build.yml │          │update-vault │ ◄─ Parallel
  └──────┬───┘          └──────┬──────┘
         │                     │
         └─────────┬───────────┘
                   ▼
            ┌─────────────┐
            │ deploy.yml  │ ◄─ Deploy to ArgoCD/Spinnaker
            └──────┬──────┘
                   │
    ┌──────────────┴──────────────┐
    ▼                             ▼
┌──────────────────┐      ┌────────────────┐
│notify-success.yml│  OR  │notify-failure │
└─────────┬────────┘      └────────┬──────┘
          │                        │
          ▼                        ▼
      [Slack]                  [Slack]
          │
          ▼
    ┌──────────────┐
    │clean-up-tag  │ ◄─ Delete tag if staging
    └──────────────┘
```

## 🔗 Integration

Each reusable workflow is called with `uses:` and can pass inputs/secrets:

```yaml
jobs:
  my-job:
    uses: kumparan/cicd-template/.github/workflows/prepare.yml@main
    with:
      project: BACKEND
    secrets:
      VAULT_UPDATE_TOKEN: ${{ secrets.VAULT_UPDATE_TOKEN }}
```

## 📖 Learn More

- **New to GitHub Actions?** Start with [QUICKSTART.md](QUICKSTART.md)
- **Need detailed docs?** Read [WORKFLOWS_README.md](WORKFLOWS_README.md)
- **Want real examples?** Check [USAGE_EXAMPLE.md](USAGE_EXAMPLE.md)

## 🤝 Contributing

To update these workflows:

1. Create a new branch
2. Edit workflows in `.github/workflows/`
3. Test in your repository
4. Create Pull Request to `main`
5. Merge (all repos will pick up changes on next deployment)

## 📞 Support

**Questions or issues?**
- 💬 Ask in #sre-internal Slack
- 📧 Email SRE team
- 🐛 Create GitHub issue in cicd-template repo

---

**Latest Update:** April 2026  
**Status:** Production Ready ✅
