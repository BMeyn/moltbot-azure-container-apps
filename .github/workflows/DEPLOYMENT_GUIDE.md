# GitHub Actions Deployment Guide

This guide explains how to set up and use the GitHub Actions workflow to automatically deploy MoltBot to your Azure subscription.

## Overview

The GitHub Actions workflow (`azure-dev.yml`) automates the deployment process using Azure Developer CLI (azd). It will:

1. Provision all required Azure infrastructure (Container Apps, Container Registry, etc.)
2. Build the MoltBot container image
3. Deploy the application to Azure Container Apps

## Prerequisites

Before setting up the GitHub Actions workflow, you need:

- ✅ An Azure subscription with Contributor access
- ✅ GitHub repository with this code
- ✅ [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli) installed locally (for initial setup)
- ✅ OpenRouter API key ([get one here](https://openrouter.ai/keys))
- ✅ Discord bot token ([create a bot here](https://discord.com/developers/applications))

## Setup Instructions

### Step 1: Create Azure Service Principal

You need to create a service principal that GitHub Actions will use to authenticate with Azure.

#### Option A: Using Federated Credentials (Recommended)

Federated credentials are more secure as they don't require storing secrets.

```bash
# Set your variables
SUBSCRIPTION_ID="your-subscription-id"
REPO_OWNER="BMeyn"  # Your GitHub username or org
REPO_NAME="moltbot-azure-container-apps"

# Create the service principal
az ad sp create-for-rbac \
  --name "github-moltbot-deploy" \
  --role contributor \
  --scopes /subscriptions/$SUBSCRIPTION_ID

# Save the output - you'll need the clientId and tenantId

# Create federated credential for main branch
az ad app federated-credential create \
  --id <APPLICATION_ID> \
  --parameters '{
    "name": "github-main",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:'"$REPO_OWNER/$REPO_NAME"':ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# Create federated credential for pull requests (optional)
az ad app federated-credential create \
  --id <APPLICATION_ID> \
  --parameters '{
    "name": "github-pr",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:'"$REPO_OWNER/$REPO_NAME"':pull_request",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

#### Option B: Using Client Secret

If federated credentials are not available in your Azure tenant:

```bash
az ad sp create-for-rbac \
  --name "github-moltbot-deploy" \
  --role contributor \
  --scopes /subscriptions/$SUBSCRIPTION_ID \
  --sdk-auth
```

Save the entire JSON output - you'll need it for the `AZURE_CREDENTIALS` secret.

### Step 2: Configure GitHub Secrets

Go to your GitHub repository → Settings → Secrets and variables → Actions → New repository secret.

#### Required Secrets

**For Azure Authentication (Option A - Federated Credentials):**
- `AZURE_CLIENT_ID`: The `clientId` from the service principal creation
- `AZURE_TENANT_ID`: The `tenantId` from the service principal creation
- `AZURE_SUBSCRIPTION_ID`: Your Azure subscription ID

**For Azure Authentication (Option B - Client Secret):**
- `AZURE_CREDENTIALS`: The full JSON output from the service principal creation
- `AZURE_SUBSCRIPTION_ID`: Your Azure subscription ID

**For Deployment Configuration:**
- `AZURE_ENV_NAME`: Environment name (e.g., `prod`, `dev`, `staging`)
- `AZURE_LOCATION`: Azure region (e.g., `eastus2`, `westus2`)

**For MoltBot Configuration:**
- `OPENROUTER_API_KEY`: Your OpenRouter API key (starts with `sk-or-v1-`)
- `DISCORD_BOT_TOKEN`: Your Discord bot token
- `DISCORD_ALLOWED_USERS`: Your Discord user ID (comma-separated for multiple users)

#### Optional Secrets

- `MOLTBOT_MODEL`: Model to use (default: `openrouter/anthropic/claude-3.5-sonnet`)
- `MOLTBOT_PERSONA_NAME`: Bot persona name (default: `Clawd`)
- `ALLOWED_IP_RANGES`: Comma-separated CIDR blocks for IP restrictions (e.g., `1.2.3.4/32`)
- `ALERT_EMAIL_ADDRESS`: Email address for Azure Monitor alerts

### Step 3: Get Your Discord User ID

1. In Discord: Settings → Advanced → Enable **Developer Mode**
2. Right-click your username → **Copy User ID**
3. Use this ID for the `DISCORD_ALLOWED_USERS` secret

### Step 4: Trigger the Workflow

The workflow will automatically run when:
- You push to the `main` branch
- You create a pull request to `main`
- You manually trigger it from the Actions tab

To manually trigger:
1. Go to your repository on GitHub
2. Click the **Actions** tab
3. Select **Deploy to Azure** workflow
4. Click **Run workflow**
5. Select the branch and click **Run workflow**

## Workflow Behavior

### On Push to Main
- Provisions infrastructure (if not exists)
- Builds and deploys the application
- Updates existing deployment with new changes

### On Pull Request
- Provisions infrastructure (if not exists)
- Builds and deploys to test the changes
- Does NOT affect production deployment

### Manual Trigger
- Same as push to main
- Useful for redeployment or troubleshooting

## Monitoring Deployment

### View Workflow Progress

1. Go to your repository → **Actions** tab
2. Click on the running workflow
3. Expand each step to see detailed logs

### Check Deployment Status

After deployment completes, the workflow summary will show:
- Environment name
- Azure location
- Subscription ID
- Application URL (if available)

### Verify Application

Once deployed, you can verify the application is running:

1. **Via Azure Portal:**
   - Go to the resource group (e.g., `rg-prod`)
   - Find the Container App
   - Check the "Live logs" or "Revision management"

2. **Via Discord:**
   - Invite your bot to a Discord server
   - Send a DM to the bot
   - You should get a response within a few seconds

## Updating Configuration

### Change Application Settings

To update environment variables like the model or persona name:

1. Update the GitHub secret (e.g., `MOLTBOT_MODEL`)
2. Re-run the workflow (push to main or manual trigger)
3. The workflow will redeploy with new settings

### Update Discord Allowed Users

1. Update the `DISCORD_ALLOWED_USERS` secret with comma-separated IDs
2. Re-run the workflow

### Rotate API Keys

1. Update the secret (e.g., `OPENROUTER_API_KEY`)
2. Re-run the workflow
3. The new key will be deployed securely

## Troubleshooting

### Workflow Fails at "Log in with Azure"

**Problem:** Authentication failed

**Solutions:**
- Verify `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, and `AZURE_SUBSCRIPTION_ID` are correct
- If using client secret, verify `AZURE_CREDENTIALS` JSON is valid
- Check that the service principal has Contributor role on the subscription

### Workflow Fails at "Provision infrastructure"

**Problem:** Resource creation failed

**Solutions:**
- Check if you have quota in the selected region
- Verify the location name is correct (e.g., `eastus2` not `East US 2`)
- Check Azure Portal for more detailed error messages

### Workflow Fails at "Deploy application"

**Problem:** Container deployment failed

**Solutions:**
- Check if the container image was built successfully
- Verify all required secrets are set
- Check Container App logs in Azure Portal

### Bot Doesn't Respond in Discord

**Problem:** Bot is deployed but not responding

**Solutions:**
- Check `DISCORD_BOT_TOKEN` is correct
- Verify `DISCORD_ALLOWED_USERS` includes your user ID
- Check Container App logs for Discord connection errors
- Ensure you invited the bot to a server before DMing

### "Unknown model" Error

**Problem:** Model configuration is incorrect

**Solutions:**
- Verify `MOLTBOT_MODEL` uses exact format: `openrouter/anthropic/claude-3.5-sonnet`
- Check [OpenRouter Models](https://openrouter.ai/models) for valid model IDs

## Cost Management

The workflow provisions resources that incur costs:
- Container Apps: ~$30-50/month (always running)
- Container Registry: ~$5/month
- Log Analytics: ~$2-5/month

**To reduce costs:**
- Scale down to 0 replicas when not in use (breaks Discord connection)
- Use a cheaper model
- Delete the resource group when not needed

## Security Best Practices

1. **Never commit secrets** to the repository - always use GitHub Secrets
2. **Use federated credentials** instead of client secrets when possible
3. **Enable IP restrictions** via `ALLOWED_IP_RANGES` secret
4. **Set up email alerts** via `ALERT_EMAIL_ADDRESS` secret
5. **Regularly rotate** API keys and bot tokens
6. **Review** Container App logs regularly for suspicious activity

## Advanced Configuration

### Deploy to Multiple Environments

Create separate workflows for different environments:

1. Copy `azure-dev.yml` to `azure-dev-staging.yml`
2. Change the branch trigger to `develop`
3. Use different environment names (e.g., `AZURE_ENV_NAME: staging`)

### Custom Build Steps

The workflow uses the postprovision hook to build the container image. To customize:

1. Edit `scripts/postprovision.sh`
2. Add custom build steps or arguments
3. Commit and push changes

### Enable Internal-Only Access

To deploy without public ingress:

1. Add secret: `INTERNAL_ONLY: "true"`
2. Update workflow to pass this parameter
3. Access only via VNet-integrated resources

## Support

For issues with:
- **GitHub Actions workflow**: Check this guide and workflow logs
- **Azure deployment**: Check [Azure Developer CLI docs](https://learn.microsoft.com/azure/developer/azure-developer-cli/)
- **MoltBot application**: Check [MoltBot documentation](https://docs.molt.bot)

## Next Steps

After successful deployment:
1. ✅ Invite your Discord bot to a server
2. ✅ Test by sending a DM
3. ✅ Monitor logs in Azure Portal
4. ✅ Set up additional users if needed
5. ✅ Configure IP restrictions for security
6. ✅ Enable email alerts for monitoring
