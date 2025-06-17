# GitHub Webhook Configuration for Jenkins

## Prerequisites
- Admin access to your GitHub repository
- Jenkins server accessible from the internet (or GitHub)
- Jenkins GitHub plugin installed

## Step 1: Configure Jenkins GitHub Plugin

### Install Required Plugins
1. Go to **Manage Jenkins** → **Manage Plugins**
2. Install these plugins if not already installed:
   - **GitHub Plugin**
   - **GitHub Integration Plugin**
   - **Generic Webhook Trigger Plugin** (you already have this)

### Configure GitHub Server in Jenkins
1. Go to **Manage Jenkins** → **Configure System**
2. Scroll to **GitHub** section
3. Click **Add GitHub Server**
4. Configure:
   - **Name**: `GitHub Server`
   - **API URL**: `https://api.github.com` (default)
   - **Credentials**: Add your GitHub PAT (Personal Access Token)

## Step 2: Create GitHub Personal Access Token

1. Go to GitHub → **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)**
2. Click **Generate new token (classic)**
3. Configure the token:
   - **Note**: `Jenkins Webhook Access`
   - **Expiration**: Choose appropriate duration
   - **Scopes**: Select these permissions:
     - `repo` (Full control of private repositories)
     - `admin:repo_hook` (Full control of repository hooks)
     - `user:email` (Access user email addresses)

4. Copy the generated token and save it securely

## Step 3: Add GitHub Credentials to Jenkins

1. Go to **Manage Jenkins** → **Manage Credentials**
2. Select the appropriate domain (usually "Global")
3. Click **Add Credentials**
4. Configure:
   - **Kind**: Secret text
   - **Scope**: Global
   - **Secret**: Paste your GitHub PAT
   - **ID**: `github_sync_bot_PAT` (matches your pipeline)
   - **Description**: `GitHub PAT for repository synchronization`

## Step 4: Configure GitHub Repository Webhook

1. Go to your GitHub repository
2. Navigate to **Settings** → **Webhooks**
3. Click **Add webhook**
4. Configure the webhook:

### Webhook Configuration
```
Payload URL: https://your-jenkins-server.com/github-webhook/
Content type: application/json
Secret: (leave empty for now, or add a secret if you want extra security)
```

### Which events would you like to trigger this webhook?
Select **"Let me select individual events"** and choose:
- ✅ **Pushes** (main trigger for your sync)
- ✅ **Branch or tag creation**
- ✅ **Branch or tag deletion** (optional, if you want to sync branch operations)

### Active
Make sure the webhook is **Active** (checked)

## Step 5: Test the Webhook

### Method 1: Use GitHub's Test Feature
1. After creating the webhook, click on it
2. Go to the **Recent Deliveries** tab
3. Click **Redeliver** on any delivery, or make a test push

### Method 2: Make a Test Commit
1. Make a small change to your repository
2. Commit and push to the `main` branch
3. Check Jenkins for a triggered build

## Step 6: Verify Jenkins Configuration

### Check Your Pipeline Configuration
Make sure your Jenkins pipeline has the correct trigger configuration:

```groovy
triggers {
    githubPush()
    // Your GitLab trigger configuration...
}
```

### Webhook URL Variations
Depending on your Jenkins setup, you might need to use one of these URLs:

**Standard GitHub Plugin:**
```
https://your-jenkins-server.com/github-webhook/
```

**If using Blue Ocean or different context:**
```
https://your-jenkins-server.com/job/YOUR_JOB_NAME/build?token=YOUR_BUILD_TOKEN
```

**Generic Webhook Trigger (alternative approach):**
```
https://your-jenkins-server.com/generic-webhook-trigger/invoke?token=github-webhook-secret
```

## Step 7: Security Considerations

### Add Webhook Secret (Recommended)
1. Generate a random secret: `openssl rand -hex 20`
2. Add it to both:
   - GitHub webhook configuration (Secret field)
   - Jenkins credentials (create new secret text credential)
3. Reference it in your pipeline if using Generic Webhook Trigger

### IP Allowlisting
Consider restricting webhook access to GitHub's IP ranges:
```
192.30.252.0/22
185.199.108.0/22
140.82.112.0/20
143.55.64.0/20
```

## Nuances
### Step 4: Jenkins security
We have chosen to use a webhook that will trigger a Jenkins synchronizing job. Configured without care, this method could expose Jenkins master to critical security threats. **Always** protect traffic and connection between the code hosting side and Jenkins masters. There are several obligatory actions to perform to secure Jenkins in production environment like securing networks with proxies, firewalls and IP lists, using mutual authorization, regular secret rotations etc.

For our laboratory purposes, we are running Jenkins master on a local machine, exposed on a local port. To allow the webhook to reach a machine, there are two classes of solutions available:
- Running secure tunnel service, allowing to tunnel a traffic to a local machine in a secure manner: [nginx](https://ngrok.com/our-product/secure-tunnels), [localtunnel](https://theboroer.github.io/localtunnel-www/), etc
- Running a webhook relay service like [Webhook Relay](https://webhookrelay.com/) that creates a public endpoint to receive webhook and it redirects it to the local machine.

I am using a Webhook relay to do the redirection:
```bash
# First, sign up on webhookrelay.com
# Create token and secret key

# Second, install CLI commands
curl -s https://webhookrelay.com/install.sh | bash

# Authenticate
relay login -k <token-key> -s <secret-key>

# Expose a local host with Jenkins to github webhooks
relay forward --bucket github-jenkins http://localhost:8080/github-webhook/
```