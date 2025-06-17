# Setting Up a GitLab Agent to detect main branch changes

There are no Apps on GitLab, so we need to create a dedicated service account:

1. Create a service account:
   - Sign up for a new GitLab account to use specifically for this automation
   - Use a descriptive username like "repo-sync-bot" or "jenkins-sync"

2. Generate a Personal Access Token (PAT):
    - Log in as the service account
    - Go to User Settings (top-right profile icon) → Access Tokens
    - Create a new token with:
      - Name: "Repository Synchronization"
      - Expiration: Set an expiration date or make it non-expiring (consider security implications)
      - Scopes: Select api, read_repository, and write_repository
    - Click "Create personal access token"
    Important: Copy and securely store the token immediately as it won't be shown again

3. Add the service account to your GitLab project:
   - Navigate to your GitLab project
   - Go to Settings → Members
   - Add the service account as a member with "Maintainer" role. Give only this user a possibility to push and merge to the protected main branch.

4. Set up webhook for GitLab to Jenkins (if you want bidirectional sync):
   - See [here](gitlab_webhook_setup.md) for details

5. Store credentials in Jenkins:
   - Add the GitLab PAT to Jenkins credentials store (as a secret text)
   - Add the Gitlab webhook secret to Jenkins credentials store (as a secret text)
   - Use Jenkins' secret management to secure this token

    Test the access:
        Verify the service account can push to the main branch
        Verify regular developers cannot push directly to main

This setup creates a dedicated GitLab service account with the minimum necessary permissions to perform the repository synchronization tasks. The account will have exclusive rights to push directly to the main branch, while regular development will need to go through merge requests.