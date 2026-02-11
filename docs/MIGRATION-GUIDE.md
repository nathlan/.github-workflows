# Migration Guide: App Registration to User-Assigned Managed Identities

This guide helps you migrate from using Azure App Registration (Service Principal) to User-Assigned Managed Identities (UAMIs) with federated credentials.

## Why Migrate?

### Current Approach (App Registration)
- ❌ Single identity for both plan and apply (no separation of duties)
- ❌ Higher risk if credentials are compromised
- ❌ Requires managing service principal lifecycle
- ⚠️ All operations tracked under same identity

### New Approach (Dual UAMIs)
- ✅ Separate identities with least privilege (Reader for plan, Owner for apply)
- ✅ Defense in depth - compromised plan job cannot modify infrastructure
- ✅ Better audit trail with separate identity actions
- ✅ Cleaner authentication without service principal secrets
- ✅ Follows Azure best practices for GitHub Actions

## Migration Steps

### Step 1: Understand Your Current Setup

Your current workflow likely uses:
```yaml
secrets:
  AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}  # Service Principal
  AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
  AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

### Step 2: Create User-Assigned Managed Identities

Follow the [Azure Managed Identity Setup Guide](./AZURE-MANAGED-IDENTITY-SETUP.md) to create:
1. Plan identity with Reader role
2. Apply identity with Owner role
3. Federated credentials for both identities

Quick CLI setup:
```bash
# Variables
RG_NAME="rg-github-identities"
LOCATION="uksouth"
GITHUB_ORG="your-org"
GITHUB_REPO="your-repo"
ENVIRONMENT="production"
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

# Create resource group
az group create --name $RG_NAME --location $LOCATION

# Create identities
az identity create --resource-group $RG_NAME --name uami-terraform-plan
az identity create --resource-group $RG_NAME --name uami-terraform-apply

# Get IDs
PLAN_CLIENT_ID=$(az identity show --resource-group $RG_NAME --name uami-terraform-plan --query clientId -o tsv)
PLAN_PRINCIPAL_ID=$(az identity show --resource-group $RG_NAME --name uami-terraform-plan --query principalId -o tsv)
APPLY_CLIENT_ID=$(az identity show --resource-group $RG_NAME --name uami-terraform-apply --query clientId -o tsv)
APPLY_PRINCIPAL_ID=$(az identity show --resource-group $RG_NAME --name uami-terraform-apply --query principalId -o tsv)

# Assign roles
az role assignment create --assignee $PLAN_PRINCIPAL_ID --role Reader --scope /subscriptions/$SUBSCRIPTION_ID
az role assignment create --assignee $APPLY_PRINCIPAL_ID --role Owner --scope /subscriptions/$SUBSCRIPTION_ID

# Create federated credentials
az identity federated-credential create \
  --resource-group $RG_NAME \
  --identity-name uami-terraform-plan \
  --name github-plan-cred \
  --issuer https://token.actions.githubusercontent.com \
  --subject "repo:$GITHUB_ORG/$GITHUB_REPO:environment:$ENVIRONMENT" \
  --audience api://AzureADTokenExchange

az identity federated-credential create \
  --resource-group $RG_NAME \
  --identity-name uami-terraform-apply \
  --name github-apply-cred \
  --issuer https://token.actions.githubusercontent.com \
  --subject "repo:$GITHUB_ORG/$GITHUB_REPO:environment:$ENVIRONMENT" \
  --audience api://AzureADTokenExchange

# Display client IDs (add these to GitHub secrets)
echo "AZURE_CLIENT_ID_PLAN: $PLAN_CLIENT_ID"
echo "AZURE_CLIENT_ID_APPLY: $APPLY_CLIENT_ID"
```

### Step 3: Update GitHub Secrets

In your GitHub repository:

1. Go to **Settings** → **Environments** → Select your environment (e.g., `production`)
2. **Add** these new secrets:
   - `AZURE_CLIENT_ID_PLAN`: Plan identity client ID
   - `AZURE_CLIENT_ID_APPLY`: Apply identity client ID
3. **Keep** existing secrets (for rollback):
   - `AZURE_TENANT_ID`
   - `AZURE_SUBSCRIPTION_ID`
   - `AZURE_CLIENT_ID` (keep temporarily for rollback)

### Step 4: Update Workflow Configuration

**Before (Old):**
```yaml
jobs:
  deploy:
    uses: nathlan/.github-workflows/.github/workflows/azure-terraform-deploy.yml@v1
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

**After (New):**
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

### Step 5: Test the Migration

1. Create a test PR to trigger terraform plan with the Reader identity
2. Verify the plan job succeeds and generates a plan
3. Merge to main (or manually trigger) to test terraform apply with the Owner identity
4. Verify the apply job succeeds and deploys resources

Test workflow example:
```yaml
name: Test UAMI Authentication
on:
  workflow_dispatch:

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  test-dual-identity:
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

### Step 6: Verify Audit Trail

Check Azure Activity Logs to verify:
1. Plan operations are performed by the Reader identity
2. Apply operations are performed by the Owner identity
3. No unauthorized operations occurred during migration

Azure CLI command:
```bash
# View recent activity by each identity
az monitor activity-log list --caller $PLAN_CLIENT_ID --max-events 10
az monitor activity-log list --caller $APPLY_CLIENT_ID --max-events 10
```

### Step 7: Clean Up Old Resources (Optional)

After successful migration and testing:

1. **Remove old App Registration** (if not used elsewhere):
   ```bash
   az ad app delete --id <old-app-id>
   ```

2. **Remove old GitHub secret** `AZURE_CLIENT_ID` from your environment

3. **Document the change** in your repository's documentation

## Rollback Plan

If you encounter issues during migration:

### Quick Rollback Steps

1. **Revert workflow changes** back to using `AZURE_CLIENT_ID`
2. **Verify** the old app registration is still functional
3. **Investigate** the issue with managed identities
4. **Retry** migration after resolving issues

### Common Issues During Migration

#### Issue: "AADSTS70021: No matching federated identity record found"

**Solution:**
- Verify federated credential subject matches exactly
- Check environment name in GitHub matches federated credential
- Ensure `id-token: write` permission is set

#### Issue: "Insufficient permissions to complete operation"

**Solution:**
- Verify role assignments for both identities
- Wait 5-10 minutes for role propagation
- Check scope (subscription vs resource group)

#### Issue: "Terraform state lock error"

**Solution:**
- Ensure both identities have access to backend storage account
- Add both identities to storage account with "Storage Blob Data Contributor" role
  ```bash
  az role assignment create \
    --assignee $PLAN_PRINCIPAL_ID \
    --role "Storage Blob Data Contributor" \
    --scope /subscriptions/$SUBSCRIPTION_ID/resourceGroups/<state-rg>/providers/Microsoft.Storage/storageAccounts/<state-storage>
  
  az role assignment create \
    --assignee $APPLY_PRINCIPAL_ID \
    --role "Storage Blob Data Contributor" \
    --scope /subscriptions/$SUBSCRIPTION_ID/resourceGroups/<state-rg>/providers/Microsoft.Storage/storageAccounts/<state-storage>
  ```

## Benefits After Migration

Once migrated, you'll enjoy:

✅ **Enhanced Security**: Separate identities with least privilege
✅ **Better Compliance**: Clear separation of read vs write operations
✅ **Improved Audit**: Distinct audit trails for plan and apply
✅ **Defense in Depth**: Compromised plan cannot modify infrastructure
✅ **Azure Best Practice**: Aligned with Microsoft recommendations

## Migration Checklist

Use this checklist to track your migration progress:

- [ ] Created plan identity (Reader role)
- [ ] Created apply identity (Owner role)
- [ ] Configured federated credentials for both identities
- [ ] Added storage account permissions for state management
- [ ] Added `AZURE_CLIENT_ID_PLAN` secret to GitHub environment
- [ ] Added `AZURE_CLIENT_ID_APPLY` secret to GitHub environment
- [ ] Updated workflow to use new secrets
- [ ] Tested plan operation with Reader identity
- [ ] Tested apply operation with Owner identity
- [ ] Verified audit trail shows separate identities
- [ ] Documented migration in repository
- [ ] Removed old app registration (if no longer needed)
- [ ] Removed old `AZURE_CLIENT_ID` secret

## Need Help?

If you encounter issues during migration:
1. Review the [Azure Managed Identity Setup Guide](./AZURE-MANAGED-IDENTITY-SETUP.md)
2. Check the troubleshooting section in the setup guide
3. Review Azure Activity Logs for authentication errors
4. Contact the platform engineering team

## Additional Resources

- [Azure Managed Identity Setup Guide](./AZURE-MANAGED-IDENTITY-SETUP.md)
- [Microsoft Docs: Workload Identity Federation](https://learn.microsoft.com/en-us/azure/active-directory/develop/workload-identity-federation)
- [GitHub Docs: OIDC with Azure](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-azure)
- [Terraform Azure Provider: Managed Identity](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/guides/managed_service_identity.html)
