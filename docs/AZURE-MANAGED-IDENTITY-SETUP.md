# Azure User-Assigned Managed Identity Setup Guide

This guide explains how to set up User-Assigned Managed Identities (UAMIs) with federated credentials for secure GitHub Actions authentication to Azure.

## Overview

The Azure Terraform Deploy workflow uses a **dual-identity security model** to implement the principle of least privilege:

- **Plan Identity**: Has Reader role for read-only access during `terraform plan`
- **Apply Identity**: Has Owner role for full access during `terraform apply`

This approach ensures that even if a plan job is compromised, it cannot modify Azure infrastructure.

## Prerequisites

- Azure subscription with appropriate permissions to create identities and role assignments
- GitHub repository where you'll use the reusable workflow
- Azure CLI or Azure Portal access
- Terraform (for infrastructure-as-code approach)

## Authentication Flow

```
GitHub Actions Job (with id-token: write permission)
  ↓
GitHub OIDC Token issued
  ↓
Azure validates token against federated credential
  ↓
Azure issues access token for User-Assigned Managed Identity
  ↓
Terraform uses identity to authenticate with Azure
```

## Setup Options

You can set up the managed identities using either:
1. Azure CLI (manual setup)
2. Terraform (infrastructure-as-code)
3. Azure Portal (UI-based)

---

## Option 1: Azure CLI Setup

### Step 1: Create Resource Group (if needed)

```bash
az group create \
  --name rg-github-identities \
  --location uksouth
```

### Step 2: Create User-Assigned Managed Identities

```bash
# Create plan identity (Reader role)
az identity create \
  --resource-group rg-github-identities \
  --name uami-github-terraform-plan

# Create apply identity (Owner role)
az identity create \
  --resource-group rg-github-identities \
  --name uami-github-terraform-apply
```

### Step 3: Get Client IDs and Principal IDs

```bash
# Plan identity
PLAN_CLIENT_ID=$(az identity show \
  --resource-group rg-github-identities \
  --name uami-github-terraform-plan \
  --query clientId -o tsv)

PLAN_PRINCIPAL_ID=$(az identity show \
  --resource-group rg-github-identities \
  --name uami-github-terraform-plan \
  --query principalId -o tsv)

# Apply identity
APPLY_CLIENT_ID=$(az identity show \
  --resource-group rg-github-identities \
  --name uami-github-terraform-apply \
  --query clientId -o tsv)

APPLY_PRINCIPAL_ID=$(az identity show \
  --resource-group rg-github-identities \
  --name uami-github-terraform-apply \
  --query principalId -o tsv)

echo "Plan Client ID: $PLAN_CLIENT_ID"
echo "Apply Client ID: $APPLY_CLIENT_ID"
```

### Step 4: Assign Roles

```bash
# Get subscription ID
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

# Assign Reader role to plan identity
az role assignment create \
  --assignee $PLAN_PRINCIPAL_ID \
  --role Reader \
  --scope /subscriptions/$SUBSCRIPTION_ID

# Assign Owner role to apply identity
az role assignment create \
  --assignee $APPLY_PRINCIPAL_ID \
  --role Owner \
  --scope /subscriptions/$SUBSCRIPTION_ID
```

**Note**: You can scope roles to specific resource groups instead of the entire subscription:
```bash
az role assignment create \
  --assignee $PLAN_PRINCIPAL_ID \
  --role Reader \
  --scope /subscriptions/$SUBSCRIPTION_ID/resourceGroups/your-rg-name
```

### Step 5: Create Federated Credentials

Replace `<org>` and `<repo>` with your GitHub organization and repository names:

```bash
# Federated credential for plan identity
az identity federated-credential create \
  --resource-group rg-github-identities \
  --identity-name uami-github-terraform-plan \
  --name github-plan-credential \
  --issuer https://token.actions.githubusercontent.com \
  --subject "repo:<org>/<repo>:environment:production" \
  --audience api://AzureADTokenExchange

# Federated credential for apply identity
az identity federated-credential create \
  --resource-group rg-github-identities \
  --identity-name uami-github-terraform-apply \
  --name github-apply-credential \
  --issuer https://token.actions.githubusercontent.com \
  --subject "repo:<org>/<repo>:environment:production" \
  --audience api://AzureADTokenExchange
```

**Subject Filter Options**:
- Environment: `repo:<org>/<repo>:environment:<environment-name>`
- Branch: `repo:<org>/<repo>:ref:refs/heads/<branch-name>`
- Pull request: `repo:<org>/<repo>:pull_request`
- Tag: `repo:<org>/<repo>:ref:refs/tags/<tag-name>`

---

## Option 2: Terraform Setup

Create a Terraform configuration to manage the identities:

```hcl
# variables.tf
variable "github_org" {
  description = "GitHub organization name"
  type        = string
}

variable "github_repo" {
  description = "GitHub repository name"
  type        = string
}

variable "environment_name" {
  description = "Environment name for federated credential"
  type        = string
  default     = "production"
}

variable "subscription_scope" {
  description = "Subscription scope for role assignments"
  type        = string
}

# main.tf
resource "azurerm_resource_group" "identities" {
  name     = "rg-github-identities"
  location = "uksouth"
}

# Plan Identity (Reader role)
resource "azurerm_user_assigned_identity" "plan" {
  name                = "uami-github-terraform-plan"
  resource_group_name = azurerm_resource_group.identities.name
  location            = azurerm_resource_group.identities.location
}

resource "azurerm_role_assignment" "plan_reader" {
  principal_id         = azurerm_user_assigned_identity.plan.principal_id
  role_definition_name = "Reader"
  scope                = var.subscription_scope
}

resource "azurerm_federated_identity_credential" "plan" {
  name                = "github-plan-credential"
  resource_group_name = azurerm_resource_group.identities.name
  parent_id           = azurerm_user_assigned_identity.plan.id

  audience = ["api://AzureADTokenExchange"]
  issuer   = "https://token.actions.githubusercontent.com"
  subject  = "repo:${var.github_org}/${var.github_repo}:environment:${var.environment_name}"
}

# Apply Identity (Owner role)
resource "azurerm_user_assigned_identity" "apply" {
  name                = "uami-github-terraform-apply"
  resource_group_name = azurerm_resource_group.identities.name
  location            = azurerm_resource_group.identities.location
}

resource "azurerm_role_assignment" "apply_owner" {
  principal_id         = azurerm_user_assigned_identity.apply.principal_id
  role_definition_name = "Owner"
  scope                = var.subscription_scope
}

resource "azurerm_federated_identity_credential" "apply" {
  name                = "github-apply-credential"
  resource_group_name = azurerm_resource_group.identities.name
  parent_id           = azurerm_user_assigned_identity.apply.id

  audience = ["api://AzureADTokenExchange"]
  issuer   = "https://token.actions.githubusercontent.com"
  subject  = "repo:${var.github_org}/${var.github_repo}:environment:${var.environment_name}"
}

# Outputs
output "plan_client_id" {
  description = "Client ID for plan identity (add to AZURE_CLIENT_ID_PLAN secret)"
  value       = azurerm_user_assigned_identity.plan.client_id
}

output "apply_client_id" {
  description = "Client ID for apply identity (add to AZURE_CLIENT_ID_APPLY secret)"
  value       = azurerm_user_assigned_identity.apply.client_id
}

output "tenant_id" {
  description = "Azure tenant ID (add to AZURE_TENANT_ID secret)"
  value       = azurerm_user_assigned_identity.plan.tenant_id
}
```

Deploy:
```bash
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

---

## Option 3: Azure Portal Setup

### Step 1: Create User-Assigned Managed Identities

1. Navigate to Azure Portal → **Managed Identities**
2. Click **+ Create**
3. Fill in details:
   - **Subscription**: Select your subscription
   - **Resource group**: Create new or select existing
   - **Region**: uksouth (or your preferred region)
   - **Name**: `uami-github-terraform-plan`
4. Click **Review + create** → **Create**
5. Repeat for `uami-github-terraform-apply`

### Step 2: Assign Roles

For each identity:
1. Navigate to the identity in Azure Portal
2. Click **Azure role assignments** → **+ Add role assignment**
3. For plan identity:
   - **Scope**: Subscription (or Resource group)
   - **Role**: Reader
4. For apply identity:
   - **Scope**: Subscription (or Resource group)
   - **Role**: Owner

### Step 3: Create Federated Credentials

For each identity:
1. Navigate to the identity in Azure Portal
2. Click **Federated credentials** → **+ Add credential**
3. Select **GitHub Actions deploying Azure resources**
4. Fill in details:
   - **Organization**: Your GitHub org
   - **Repository**: Your repo name
   - **Entity type**: Environment
   - **Environment name**: production (or your environment)
   - **Name**: `github-plan-credential` or `github-apply-credential`
5. Click **Add**

### Step 4: Get Client IDs

For each identity:
1. Navigate to identity → **Overview**
2. Copy **Client ID** (you'll need this for GitHub secrets)

---

## GitHub Repository Configuration

### Step 1: Create Environment

1. Go to your repository → **Settings** → **Environments**
2. Click **New environment**
3. Name: `production` (or match your federated credential subject)
4. Configure protection rules:
   - ✅ Required reviewers (recommended)
   - ✅ Wait timer (optional)

### Step 2: Add Secrets to Environment

Add the following secrets to your environment:

- `AZURE_CLIENT_ID_PLAN`: Client ID from plan identity
- `AZURE_CLIENT_ID_APPLY`: Client ID from apply identity
- `AZURE_TENANT_ID`: Your Azure tenant ID
- `AZURE_SUBSCRIPTION_ID`: Your Azure subscription ID

**Note**: Client IDs are NOT sensitive - they're identifiers, not credentials. The security comes from the federated credential trust relationship.

### Step 3: Configure Workflow Permissions

Ensure your workflow has the `id-token: write` permission (already set in the reusable workflow).

---

## Testing the Setup

Create a test workflow in your repository:

```yaml
name: Test Azure Authentication

on:
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  test-plan-identity:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Azure Login (Plan Identity)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID_PLAN }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Test Azure CLI
        run: |
          az account show
          az group list --output table

  test-apply-identity:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Azure Login (Apply Identity)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID_APPLY }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Test Azure CLI
        run: |
          az account show
          az group list --output table
```

---

## Troubleshooting

### Error: "AADSTS70021: No matching federated identity record found"

**Cause**: The federated credential subject doesn't match the GitHub context.

**Solution**:
1. Check your federated credential subject matches exactly
2. Verify environment name if using environment-based subject
3. Check branch name if using ref-based subject

Example subjects:
- Environment: `repo:myorg/myrepo:environment:production`
- Branch: `repo:myorg/myrepo:ref:refs/heads/main`
- PR: `repo:myorg/myrepo:pull_request`

### Error: "Authorization failed"

**Cause**: Identity doesn't have required permissions.

**Solution**:
1. Verify role assignment exists: `az role assignment list --assignee <principal-id>`
2. Check role scope (subscription vs resource group)
3. Wait a few minutes for role assignment to propagate

### Error: "The subscription is not registered to use namespace"

**Cause**: Required resource provider not registered.

**Solution**:
```bash
az provider register --namespace Microsoft.ManagedIdentity
az provider register --namespace Microsoft.Authorization
```

---

## Security Best Practices

✅ **Use environment-scoped subjects** for federated credentials to limit token usage to specific environments

✅ **Scope role assignments** to specific resource groups instead of entire subscription when possible

✅ **Use separate identities** for different environments (dev/staging/prod)

✅ **Enable environment protection rules** with required reviewers for production deployments

✅ **Audit regularly** using Azure Activity Logs to monitor identity usage

✅ **Rotate federated credentials** periodically by recreating them (though OIDC tokens are short-lived)

---

## Additional Resources

- [Microsoft Docs: GitHub Actions OIDC with Azure](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-identity)
- [Terraform azurerm Provider: Managed Identity Authentication](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/guides/managed_service_identity.html)
- [GitHub Docs: OIDC with Azure](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-azure)
- [Azure Samples: GitHub Terraform OIDC CI/CD](https://github.com/Azure-Samples/github-terraform-oidc-ci-cd)

---

## Summary Checklist

Before using the reusable workflow, ensure:

- [ ] Two User-Assigned Managed Identities created (plan + apply)
- [ ] Reader role assigned to plan identity
- [ ] Owner role assigned to apply identity
- [ ] Federated credentials created for both identities
- [ ] GitHub environment created
- [ ] All four secrets added to environment (two client IDs, tenant ID, subscription ID)
- [ ] Workflow permissions include `id-token: write`
- [ ] Test workflow run successful

Once complete, you can use the reusable workflow in your repository!
