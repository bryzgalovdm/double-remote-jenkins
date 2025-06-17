# GitLab Webhook Configuration for Jenkins

## Prerequisites
- Admin/Maintainer access to your GitLab repository
- Jenkins server accessible from the internet (or GitLab)
- Generic Webhook Trigger Plugin installed in Jenkins

## Step 1: Create GitLab Personal Access Token

1. Go to GitLab → **User Settings** → **Access Tokens**
2. Click **Add new token**
3. Configure the token:
   - **Token name**: `Jenkins Sync Bot`
   - **Expiration date**: Choose appropriate duration
   - **Select scopes**:
     - ✅ `api` (Access the authenticated user's API)
     - ✅ `read_repository` (Read repository)
     - ✅ `write_repository` (Write repository)

4. Click **Create personal access token**
5. Copy the generated token and save it securely

## Step 2: Add GitLab Credentials to Jenkins

1. Go to **Manage Jenkins** → **Manage Credentials**
2. Select the appropriate domain (usually "Global")
3. Click **Add Credentials**
4. Configure:
   - **Kind**: Secret text
   - **Scope**: Global
   - **Secret**: Paste your GitLab PAT
   - **ID**: `gitlab_synchro_bot_PAT` (matches your pipeline)
   - **Description**: `GitLab PAT for repository synchronization`

## Step 3: Understand Your Pipeline's GitLab Configuration

Your pipeline already has the GitLab webhook configuration:

```groovy
GenericTrigger(
    genericVariables: [
        [key: 'ref', value: '$.ref'],
        [key: 'repository_url', value: '$.repository.git_http_url'],
        [key: 'webhook_source', value: 'gitlab'] // Custom identifier
    ],
    causeString: 'Triggered by GitLab webhook',
    token: 'gitlab-webhook-secret', // Your webhook secret
    printContributedVariables: true,
    printPostContent: true
)
```

This means:
- **Webhook Token**: `gitlab-webhook-secret`
- **Webhook URL**: `https://your-jenkins-server.com/generic-webhook-trigger/invoke?token=gitlab-webhook-secret`

## Step 4: Configure GitLab Repository Webhook

1. Go to your GitLab repository
2. Navigate to **Settings** → **Webhooks**
3. Click **Add new webhook**
4. Configure the webhook:

### Webhook Configuration
```
URL: https://your-jenkins-server.com/generic-webhook-trigger/invoke?token=gitlab-webhook-secret
Secret token: gitlab-webhook-secret
Trigger: Push events
```

### Detailed Settings:

**URL**: 
```
https://your-jenkins-server.com/generic-webhook-trigger/invoke?token=gitlab-webhook-secret
```

**Secret token**: 
```
gitlab-webhook-secret
```

**Trigger events** - Select:
- ✅ **Push events** (main trigger)
- ✅ **Tag push events** (optional)
- ⚠️ **Wildcard pattern**: Leave empty or set to `main` if you only want main branch
- ⚠️ **Branch filter**: Leave empty or set to `main` to match your pipeline

**Advanced settings**:
- ✅ **Enable SSL verification** (recommended if your Jenkins has valid SSL)
- ✅ **Push events** for all branches, or specify `main` branch only

## Step 5: Test the GitLab Webhook

### Method 1: Use GitLab's Test Feature
1. After creating the webhook, you'll see it in the webhooks list
2. Click **Test** dropdown next to your webhook
3. Select **Push events**
4. Check the response - should show HTTP 200

### Method 2: Make a Test Commit
1. Make a small change to your GitLab repository
2. Commit and push to the `main` branch
3. Check Jenkins for a triggered build

### Method 3: Check Recent Deliveries
1. Go back to **Settings** → **Webhooks**
2. Click **Edit** on your webhook
3. Scroll down to **Recent events** to see delivery status

## Step 6: Webhook Payload Analysis

Your pipeline extracts these variables from GitLab's webhook payload:

```json
{
  "ref": "refs/heads/main",
  "repository": {
    "git_http_url": "https://gitlab.com/your-username/your-repo.git"
  }
}
```

The pipeline uses:
- `$.ref` → `ref` variable (branch reference)
- `$.repository.git_http_url` → `repository_url` variable
- `'gitlab'` → `webhook_source` variable (hardcoded identifier)

## Step 7: Verify Jenkins Configuration

### Check Generic Webhook Trigger Plugin
1. Go to **Manage Jenkins** → **Manage Plugins**
2. Verify **Generic Webhook Trigger Plugin** is installed and up-to-date

### Pipeline Token Security
Your pipeline uses the token `gitlab-webhook-secret`. For better security, consider:

1. **Generate a strong token**:
   ```bash
   openssl rand -hex 32
   ```

2. **Update your pipeline** with the new token:
   ```groovy
   token: 'your-new-strong-token'
   ```

3. **Update GitLab webhook** with the same token

## Step 8: Troubleshooting

### Common Issues and Solutions

#### Webhook Shows Failed Status
- **Check URL**: Ensure Jenkins is accessible from GitLab
- **Check Token**: Verify the token matches between GitLab and Jenkins pipeline
- **Check Response**: Look at the HTTP response code in GitLab's webhook logs

#### Jenkins Build Not Triggering
1. **Check Jenkins logs**: 
   - **Manage Jenkins** → **System Log**
   - Look for Generic Webhook Trigger entries

2. **Enable debug logging**:
   - Add logger for `org.jenkinsci.plugins.gwt` at DEBUG level

3. **Verify pipeline syntax**:
   - Ensure `GenericTrigger` is properly configured in your pipeline

#### Wrong Branch Triggering
Your pipeline has this logic:
```groovy
if (env.SYNC_BRANCH != params.BRANCH_NAME) {
    echo "Webhook is for branch ${env.SYNC_BRANCH}, but we only sync ${params.BRANCH_NAME}. Aborting."
    currentBuild.result = 'ABORTED'
    error("Webhook for non-monitored branch")
}
```

To limit GitLab webhooks to specific branches:
1. In GitLab webhook settings, use **Wildcard pattern**: `main`
2. Or use **Branch filter** in the webhook configuration

#### SSL Certificate Issues
If you get SSL errors:
- Either fix your Jenkins SSL certificate
- Or disable SSL verification in GitLab webhook settings (not recommended for production)

### Debug Commands
You can test your webhook URL manually:

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "X-Gitlab-Token: gitlab-webhook-secret" \
  -d '{
    "ref": "refs/heads/main",
    "repository": {
      "git_http_url": "https://gitlab.com/your-username/your-repo.git"
    }
  }' \
  "https://your-jenkins-server.com/generic-webhook-trigger/invoke?token=gitlab-webhook-secret"
```

## Step 9: Security Best Practices

### Use Strong Webhook Tokens
```bash
# Generate a strong token
openssl rand -hex 32
```

### IP Allowlisting (Optional)
Consider restricting webhook access to GitLab's IP ranges. GitLab publishes their IP ranges, but they change frequently.

### Monitor Webhook Activity
Regularly check:
- GitLab webhook delivery logs
- Jenkins build logs
- Failed webhook attempts

## Step 10: Webhook Payload Examples

### Successful Push Event Payload
```json
{
  "object_kind": "push",
  "ref": "refs/heads/main",
  "checkout_sha": "da1560886d4f094c3e6c9ef40349f7d38b5d27d7",
  "repository": {
    "name": "double_repo",
    "url": "git@gitlab.com:bryzgalov_dm/double_repo.git",
    "description": "",
    "homepage": "https://gitlab.com/bryzgalov_dm/double_repo",
    "git_http_url": "https://gitlab.com/bryzgalov_dm/double_repo.git",
    "git_ssh_url": "git@gitlab.com:bryzgalov_dm/double_repo.git"
  },
  "commits": [
    {
      "id": "da1560886d4f094c3e6c9ef40349f7d38b5d27d7",
      "message": "Update README.md",
      "timestamp": "2025-01-15T10:00:00Z",
      "author": {
        "name": "Your Name",
        "email": "your.email@example.com"
      }
    }
  ]
}
```