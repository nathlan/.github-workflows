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
      AZURE_CLIENT_ID_PLAN: ${{ secrets.AZURE_CLIENT_ID_PLAN }}
      AZURE_CLIENT_ID_APPLY: ${{ secrets.AZURE_CLIENT_ID_APPLY }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

**Features:**

- ✅ Azure OIDC authentication with User-Assigned Managed Identities (no stored credentials)
- ✅ Dual-identity security model (Reader for plan, Owner for apply)
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

This workflow uses User-Assigned Managed Identities (UAMIs) with federated credentials for secure, credential-less authentication:

- `AZURE_CLIENT_ID_PLAN` - Client ID of the UAMI with **Reader** role (for terraform plan)
- `AZURE_CLIENT_ID_APPLY` - Client ID of the UAMI with **Owner** role (for terraform apply)
- `AZURE_TENANT_ID` - Azure tenant ID
- `AZURE_SUBSCRIPTION_ID` - Azure subscription ID

**Authentication Model:**

This workflow implements a dual-identity security model following the principle of least privilege:

1. **Plan Identity (Reader Role)**: Used during `terraform plan` to assess changes. Has read-only access to Azure resources.
2. **Apply Identity (Owner Role)**: Used during `terraform apply` to deploy changes. Has full access to create, modify, and delete resources.

Each identity must be configured with federated credentials to trust your GitHub repository:
- **Issuer**: `https://token.actions.githubusercontent.com`
- **Subject**: `repo:<org>/<repo>:environment:<environment>` (or appropriate filter)
- **Audience**: `api://AzureADTokenExchange`

**Benefits:**
- ✅ No secrets stored (only client IDs, which are not sensitive)
- ✅ Least privilege access control
- ✅ Separate audit trails for plan vs apply operations
- ✅ Defense in depth - compromised plan job cannot modify infrastructure

**Required Environment:**

Create an environment in your repository (e.g., `production`) with:
- Required reviewers (platform team members)
- Environment secrets (listed above)

## Documentation

For detailed usage instructions and troubleshooting, see:
- **[Azure Managed Identity Setup Guide](docs/AZURE-MANAGED-IDENTITY-SETUP.md)** - Complete guide for setting up dual UAMIs with federated credentials
- **[Migration Guide](docs/MIGRATION-GUIDE.md)** - Step-by-step guide for migrating from App Registration to Managed Identities
- Workflow file comments in `.github/workflows/azure-terraform-deploy.yml`
- Example implementation in `nathlan/.github-private` repository

## Support

For questions or issues:
- Create an issue in this repository
- Contact the platform engineering team
- Reference the ALZ vending documentation