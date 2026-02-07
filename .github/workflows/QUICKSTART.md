# Quick Start: GitHub Actions Deployment

This is a quick reference for setting up GitHub Actions deployment. For detailed instructions, see [DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md).

## 🚀 5-Minute Setup

### 1. Create Service Principal

```bash
# Replace with your subscription ID
SUBSCRIPTION_ID="your-subscription-id"

# Create service principal
az ad sp create-for-rbac \
  --name "github-moltbot-deploy" \
  --role contributor \
  --scopes /subscriptions/$SUBSCRIPTION_ID \
  --sdk-auth
```

Save the JSON output for the next step.

### 2. Configure GitHub Secrets

Go to: **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

Add these secrets:

| Secret Name | Value | Where to Get It |
|------------|-------|-----------------|
| `AZURE_CREDENTIALS` | JSON output from step 1 | Service principal output |
| `AZURE_SUBSCRIPTION_ID` | Your Azure subscription ID | `az account show --query id -o tsv` |
| `AZURE_ENV_NAME` | `prod` | Choose any name |
| `AZURE_LOCATION` | `eastus2` | Choose any Azure region |
| `OPENROUTER_API_KEY` | `sk-or-v1-...` | [openrouter.ai/keys](https://openrouter.ai/keys) |
| `DISCORD_BOT_TOKEN` | Your bot token | [Discord Developer Portal](https://discord.com/developers/applications) |
| `DISCORD_ALLOWED_USERS` | Your Discord user ID | Right-click username → Copy User ID |

### 3. Trigger Deployment

**Option A: Push to main branch**
```bash
git push origin main
```

**Option B: Manual trigger**
1. Go to **Actions** tab
2. Select **Deploy to Azure**
3. Click **Run workflow**

### 4. Monitor Progress

- Watch the workflow run in the **Actions** tab
- Deployment takes ~5-10 minutes
- Check the summary for your app URL

### 5. Test Your Bot

1. Invite bot to a Discord server using OAuth2 URL
2. Send a DM to the bot
3. Get a response! 🎉

## Common Issues

**Authentication failed:**
- Verify all Azure secrets are set correctly
- Check service principal has Contributor role

**Bot not responding:**
- Verify `DISCORD_BOT_TOKEN` is correct
- Check `DISCORD_ALLOWED_USERS` has your user ID
- Ensure you invited bot to a server first

**Need help?** See [DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md) for detailed troubleshooting.
