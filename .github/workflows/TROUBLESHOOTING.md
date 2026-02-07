# Troubleshooting GitHub Actions Deployment

This guide helps you diagnose and fix common issues with the GitHub Actions deployment workflow.

## Workflow Validation Checklist

Before running the workflow, verify:

- [ ] All required secrets are configured in GitHub
- [ ] Service principal has correct permissions
- [ ] Azure subscription is active and has quota
- [ ] Discord bot token is valid
- [ ] OpenRouter API key has credits

## Common Errors and Solutions

### 1. Authentication Errors

#### Error: `AADSTS700016: Application with identifier was not found`

**Cause:** Service principal client ID is incorrect or doesn't exist.

**Solution:**
1. Verify `AZURE_CLIENT_ID` secret matches the service principal
2. Check the service principal exists: `az ad sp show --id <CLIENT_ID>`
3. Recreate the service principal if needed

#### Error: `AZURE_CREDENTIALS secret is not set`

**Cause:** GitHub secret is missing.

**Solution:**
1. Go to repository Settings → Secrets and variables → Actions
2. Add `AZURE_CREDENTIALS` secret with service principal JSON
3. Ensure the JSON is properly formatted

#### Error: `InvalidAuthenticationToken`

**Cause:** Service principal credentials are expired or invalid.

**Solution:**
1. Create a new service principal
2. Update all Azure-related secrets in GitHub
3. Ensure the service principal has Contributor role

### 2. Provisioning Errors

#### Error: `Location 'xyz' is not available for subscription`

**Cause:** Selected Azure region doesn't support required services.

**Solution:**
1. Change `AZURE_LOCATION` to a supported region
2. Try: `eastus2`, `westus2`, `centralus`, `westeurope`
3. Check available regions: `az account list-locations -o table`

#### Error: `The subscription is not registered to use namespace 'Microsoft.App'`

**Cause:** Container Apps provider not registered.

**Solution:**
```bash
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights
# Wait a few minutes for registration to complete
```

#### Error: `Quota exceeded for cores`

**Cause:** Not enough quota in the subscription.

**Solution:**
1. Request quota increase in Azure Portal
2. Or use a different region with available quota
3. Or delete unused resources to free up quota

### 3. Container Build Errors

#### Error: `Failed to build container image`

**Cause:** Docker build failed during postprovision hook.

**Solution:**
1. Check workflow logs for specific build error
2. Verify Dockerfile exists at `src/moltbot/Dockerfile`
3. Check if ACR has sufficient storage quota
4. Try building locally first to debug

#### Error: `Access denied to container registry`

**Cause:** Managed identity doesn't have ACR pull permission.

**Solution:**
```bash
# Get the identity and ACR IDs
IDENTITY_ID=$(az identity show --resource-group rg-<ENV> --name <IDENTITY> --query principalId -o tsv)
ACR_ID=$(az acr show --resource-group rg-<ENV> --name <ACR> --query id -o tsv)

# Grant permission
az role assignment create --assignee $IDENTITY_ID --role AcrPull --scope $ACR_ID
```

### 4. Deployment Errors

#### Error: `Container app failed to start`

**Cause:** Application crashed on startup.

**Solution:**
1. Check required secrets are set:
   - `OPENROUTER_API_KEY`
   - `DISCORD_BOT_TOKEN`
   - `DISCORD_ALLOWED_USERS`
2. View container logs in Azure Portal
3. Verify environment variables are correctly passed

#### Error: `Secret reference not found`

**Cause:** Secret not properly configured in Container App.

**Solution:**
1. Check workflow configured all secrets
2. Verify `azd env set` commands in workflow
3. Check main.parameters.json for correct variable names

### 5. Application Errors

#### Bot Doesn't Respond in Discord

**Possible Causes:**
1. Discord bot token is invalid
2. User ID not in allowlist
3. Model configuration is wrong
4. OpenRouter API key is invalid

**Debugging Steps:**

1. Check Container App logs:
```bash
az containerapp logs show \
  --name <APP_NAME> \
  --resource-group rg-<ENV> \
  --tail 50 --type console
```

2. Look for these success messages:
```
Discord channel configured: yes (DM allowlist: 123456789)
[discord] logged in to discord as 987654321
[gateway] agent model: openrouter/anthropic/claude-3.5-sonnet
```

3. Common error messages:
   - `Unknown model:` - Fix model ID format
   - `HTTP 401: authentication_error` - Invalid API key
   - `[discord] channel exited` - Invalid bot token

#### Error: `Unknown model: ...`

**Cause:** Model ID format is incorrect.

**Solution:**
1. Use exact format: `openrouter/anthropic/claude-3.5-sonnet`
2. Check [OpenRouter Models](https://openrouter.ai/models) for valid IDs
3. Update `MOLTBOT_MODEL` secret and redeploy

### 6. GitHub Actions Specific Errors

#### Error: `azd: command not found`

**Cause:** Azure Developer CLI installation step failed.

**Solution:**
1. Check the "Install Azure Developer CLI" step logs
2. Verify `Azure/setup-azd@v1.0.0` action is available
3. Try updating to latest version of the action

#### Error: `azd env set: environment not initialized`

**Cause:** azd environment wasn't created before setting variables.

**Solution:**
The workflow should automatically initialize the environment. If it doesn't:
1. Check `AZURE_ENV_NAME` secret is set
2. Verify the workflow has correct azd auth login step
3. Check workflow logs for initialization errors

#### Error: `GITHUB_OUTPUT: Permission denied`

**Cause:** Workflow doesn't have write permissions.

**Solution:**
Verify workflow has correct permissions block:
```yaml
permissions:
  id-token: write
  contents: read
```

### 7. Workflow Doesn't Trigger

#### Workflow doesn't run on push

**Possible Causes:**
1. Branch name doesn't match trigger
2. Workflow file has syntax errors
3. Actions are disabled for the repository

**Solutions:**
1. Check you're pushing to `main` branch
2. Validate YAML syntax
3. Enable Actions in repository Settings → Actions

#### Manual trigger option not visible

**Cause:** `workflow_dispatch` not configured or Actions disabled.

**Solution:**
1. Verify `workflow_dispatch` is in the `on:` section
2. Enable Actions in repository settings
3. Refresh the Actions page

## Getting More Information

### View Detailed Workflow Logs

1. Go to repository → **Actions** tab
2. Click on the failed workflow run
3. Click on the job name
4. Expand each step to see detailed output
5. Look for red X marks indicating failures

### Check Azure Resources

```bash
# List all resources in the resource group
az resource list --resource-group rg-<ENV> -o table

# Check Container App status
az containerapp show \
  --name <APP_NAME> \
  --resource-group rg-<ENV> \
  --query properties.runningStatus

# View live logs
az containerapp logs show \
  --name <APP_NAME> \
  --resource-group rg-<ENV> \
  --follow
```

### Test Service Principal Locally

```bash
# Login with service principal
az login --service-principal \
  --username <CLIENT_ID> \
  --password <CLIENT_SECRET> \
  --tenant <TENANT_ID>

# Verify permissions
az role assignment list --assignee <CLIENT_ID>
```

## Still Having Issues?

If you're still experiencing problems:

1. **Check the detailed logs** in both GitHub Actions and Azure Portal
2. **Verify all prerequisites** are met (see [QUICKSTART.md](QUICKSTART.md))
3. **Review the deployment guide** for step-by-step instructions ([DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md))
4. **Test locally with azd** to isolate GitHub Actions-specific issues
5. **Create an issue** in the repository with:
   - Error message from workflow logs
   - Azure subscription details (region, tier)
   - Steps to reproduce

## Useful Commands

### Reset Everything and Start Fresh

```bash
# Delete the resource group
az group delete --name rg-<ENV> --yes --no-wait

# Delete the azd environment locally (if testing locally)
azd env delete <ENV_NAME>

# In GitHub: Delete all secrets and recreate them
```

### Quick Health Check

```bash
# Check if Container App is running
az containerapp show \
  --name <APP_NAME> \
  --resource-group rg-<ENV> \
  --query "{Name:name, Status:properties.runningStatus, URL:properties.configuration.ingress.fqdn}"

# Check recent logs for errors
az containerapp logs show \
  --name <APP_NAME> \
  --resource-group rg-<ENV> \
  --tail 20 | grep -i error
```
