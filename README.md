# Automated Git Branch Synchronization Pipeline

## The Problem

Many development teams use branch protection rules (especially on `main` or `master`) to ensure stability and enforce code review via Pull/Merge Requests. This is standard practice and works well for a single remote repository.

However, when you need to keep a specific branch (like `main`) synchronized across *multiple* Git remotes hosted on different platforms (e.g., GitHub and GitLab), this presents a challenge:

1.  **Direct Pushes Blocked:** The same branch protection rules prevent easy synchronization via direct `git push`.
2.  **PR/MR Overhead:** Creating Pull/Merge Requests simply to sync changes between remotes adds unnecessary history noise (merge commits) and makes the histories diverge slightly, defeating the purpose of a true mirror.

## The Solution

This repository provides a solution using Jenkins CI/CD to automate the synchronization of a designated branch (typically `main`) between two or more Git remotes *after* changes have been properly merged via the standard PR/MR process on one of the remotes.

The core idea is to have a trusted, automated process that has special permission to bypass branch protection rules *only* for the purpose of synchronization.

**Key Components:**

1.  **Dedicated Credentials:** Specific accounts, tokens, or applications (e.g., a GitHub App, a dedicated GitLab user/token) are created with permissions to push directly to the protected branch on their respective platforms. These are *only* used by the automation.
2.  **Webhooks:** Each Git remote is configured to send a webhook notification to Jenkins whenever a push occurs to the target branch (`main`).
3.  **Jenkins Pipeline:** A Jenkins pipeline (`Jenkinsfile` in this repository) listens for these webhooks.
4.  **Synchronization Logic:** When triggered, the Jenkins pipeline:
    * Identifies which remote triggered the sync.
    * Fetches the latest state of the target branch from the source remote.
    * Uses the dedicated credentials for the *other* remote(s) to push this exact state directly to their target branch(es). This is typically a fast-forward push, ensuring the histories remain identical.

This effectively mirrors the state of the `main` branch from the remote where a merge just occurred to all other configured remotes.

## Example Scenario: GitHub <=> GitLab Sync

This repository provides a concrete example and documentation for synchronizing the `main` branch between a GitHub repository and a GitLab repository.

* Changes merged to `main` on GitHub trigger Jenkins to push the new `main` state to GitLab.
* Changes merged to `main` on GitLab trigger Jenkins to push the new `main` state to GitHub.

## Repository Structure

* `Jenkinsfile`: Contains the Jenkins declarative pipeline definition that handles the webhook triggering and synchronization logic.
* `docs/`: Contains detailed Markdown guides on how to configure the necessary permissions and credentials on different Git providers.
    * `docs/github_app_setup.md`: [Guide](docs/github_app_setup.md) for setting up a GitHub App with the required permissions.
    * `docs/gitlab_account_setup.md`: [Guide](docs/gitlab_account_setup.md) for setting up a dedicated GitLab user/token with the required permissions.
    * *(Add more guides here for other providers as needed)*

## Getting Started

1.  **Prerequisites:**
    * A running Jenkins instance.
    * Necessary Jenkins plugins (e.g., Pipeline, Git, Credentials Binding, HTTP Request or appropriate SCM plugins for webhook handling).
    * `git` installed on the Jenkins agent(s) that will run the pipeline.

2.  **Configure Git Providers:**
    * Follow the guides in the `docs/` folder to set up the dedicated accounts/apps/tokens on your chosen Git platforms (e.g., GitHub, GitLab).
    * Ensure these dedicated entities have *direct push access* to the target branch (`main`), bypassing any branch protection rules applied to regular users/developers.

3.  **Configure Jenkins:**
    * **Credentials:** Add the credentials (API tokens, SSH keys, etc.) for your dedicated Git provider entities to Jenkins Credentials Manager. Note the credential IDs used.
    * **Pipeline:** Create a new Jenkins Pipeline job and configure it to use the `Jenkinsfile` from this repository (via SCM, **otherwise the webhooks will not work**). You will likely need to update credential IDs and repository URLs within the `Jenkinsfile` or configure them as job parameters.
    * **Webhook Listener:** Configure Jenkins to receive webhooks from your Git providers. This might involve generic webhook triggers or specific SCM integration plugins that can parse payloads. The pipeline needs to identify the source repository from the webhook.

4.  **Configure Repositories:**
    * On each of your Git repositories (e.g., on GitHub and GitLab), configure a webhook that points to your Jenkins instance's webhook listener URL.
    * Ensure the webhook triggers only on `push` events specifically for the branch you want to synchronize (`main`).

## Extending to Other Providers

While this repository uses GitHub and GitLab as primary examples, the core pattern is adaptable to other Git hosting providers like:

* Bitbucket (Cloud or Server/Data Center)
* Azure Repos
* ...

**To adapt this solution:**

1.  **Authentication:** Determine the best method for granting dedicated, automated push access on the target provider (e.g., Access Tokens, Deploy Keys, Service Accounts, Application Passwords). Follow the provider's documentation to set this up.
2.  **Documentation:** Create a corresponding setup guide in the `docs/` directory.
3.  **Jenkins Credentials:** Store the new credentials securely in Jenkins.
4.  **Webhook Configuration:** Set up webhooks on the new provider's repository.
5.  **Pipeline Logic (`Jenkinsfile`):**
    * Update the pipeline to handle credentials for the new provider.
    * Adjust any logic needed to parse webhook payloads if they differ significantly (though often the triggering repository URL is sufficient).
    * Add logic to push to the new remote when triggered by other remotes, and vice-versa.

## Important Considerations

* **Security:** Protect the credentials used by Jenkins diligently. They grant powerful push access. Use Jenkins Credentials Manager and avoid hardcoding secrets.
* **Force Pushing:** The pipeline might need to use `git push --force` or `--force-with-lease` if branches somehow diverge. This is powerful and potentially destructive if misconfigured. Ensure the logic is sound. The primary goal of the PR-only workflow is to prevent divergence in the first place.
* **Race Conditions/Loops:** Implement checks (e.g., based on commit hashes or triggering user) in the Jenkins pipeline to prevent accidental infinite synchronization loops if webhook events fire too closely together.
* **Initial State:** Ensure the target branches on all remotes are identical *before* enabling the automation.

## Contributing

Contributions are welcome! If you adapt this solution for other providers or improve the existing pipeline/documentation, please feel free to open a Pull Request.
