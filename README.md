# Reusable GitHub Actions Workflows

This repository contains reusable GitHub Actions workflows for Terraform deployments across the organization.

## Available Workflows

### Azure Terraform Deploy

**File:** `.github/workflows/azure-terraform-deploy.yml`

**Purpose:** Reusable workflow for Azure Terraform deployments with OIDC authentication, security scanning, and approval gates.

**Usage:**

```yaml
jobs:
  deploy:
    uses: nathlan/.github-workflows/.github/workflows/azure-terraform-deploy.yml@main
    with:
      environment: production
      terraform-version: '1.9.0'
      working-directory: terraform
      azure-region: uksouth
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

**Features:**

- ✅ Azure OIDC authentication (no stored credentials)
- ✅ Security scanning with Checkov (fails on violations)
- ✅ TFLint validation
- ✅ Plan artifact reuse (prevents drift)
- ✅ Environment protection with manual approvals
- ✅ Comprehensive PR comments with plan output

**Workflow Jobs:**

1. **validate** - Terraform format, validation, TFLint checks
2. **security** - Checkov security scanning with SARIF upload
3. **plan** - Generate Terraform plan (with PR comments)
4. **apply** - Deploy with approval gate (main branch only)

**Required Secrets:**

- `AZURE_CLIENT_ID` - Service principal client ID (OIDC)
- `AZURE_TENANT_ID` - Azure tenant ID
- `AZURE_SUBSCRIPTION_ID` - Azure subscription ID

**Required Environment:**

Create an environment in your repository (e.g., `production`) with:
- Required reviewers (platform team members)
- Environment secrets (listed above)

## Documentation

For detailed usage instructions and troubleshooting, see:
- Workflow file comments in `.github/workflows/azure-terraform-deploy.yml`
- Example implementation in `nathlan/.github-private` repository
- Deployment guides in `docs/DEPLOYMENT.md` (if available)

## Support

For questions or issues:
- Create an issue in this repository
- Contact the platform engineering team
- Reference the ALZ vending documentation