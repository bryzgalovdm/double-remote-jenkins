pipeline {
    agent any
    
    triggers {
        // Listen for webhooks from both GitHub and GitLab
        githubPush()
        // Generic webhook trigger will help us capture GitLab webhooks
        GenericTrigger(
            genericVariables: [
                [key: 'ref', value: '$.ref'],
                [key: 'repository_url', value: '$.repository.git_http_url'],
                [key: 'webhook_source', value: 'gitlab'] // Custom value to identify source
            ],
            causeString: 'Triggered by GitLab webhook',
            token: 'gitlab-webhook-secret', // Use the secret token you configured
            printContributedVariables: true,
            printPostContent: true
        )
    }
    
    parameters {
        string(name: 'GITHUB_REPO_URL', defaultValue: 'https://github.com/org/double-remote-repo.git', description: 'GitHub repository URL')
        string(name: 'GITLAB_REPO_URL', defaultValue: 'https://gitlab.com/org/double-remote-repo.git', description: 'GitLab repository URL')
        string(name: 'BRANCH_NAME', defaultValue: 'main', description: 'Branch to synchronize')
    }
    
    environment {
        // GitHub PAT with repo scope permissions to push to protected branches
        GITHUB_CRED_ID = 'bdm-robot-pat' 
        // GitLab PAT with write_repository scope
        GITLAB_CRED_ID = 'gitlab_synchro_bot_PAT'
    }
    
    stages {
        stage('Determine Source and Target') {
            steps {
                script {
                    // Default to GitHub as source (will be overridden if we detect GitLab webhook)
                    env.SOURCE_REPO = params.GITHUB_REPO_URL
                    env.TARGET_REPO = params.GITLAB_REPO_URL
                    env.SOURCE_CRED_ID = env.GITHUB_CRED_ID
                    env.TARGET_CRED_ID = env.GITLAB_CRED_ID
                    env.SOURCE_NAME = 'GitHub'
                    env.TARGET_NAME = 'GitLab'
                    
                    // Get all causes for this build
                    def causes = currentBuild.getBuildCauses()
                    echo "Build causes: ${causes}"
                    
                    // Check for GitLab webhook
                    if (env.webhook_source == 'gitlab' || causes.toString().contains('GitLab')) {
                        echo "Detected webhook from GitLab"
                        // Swap source and target
                        env.SOURCE_REPO = params.GITLAB_REPO_URL
                        env.TARGET_REPO = params.GITHUB_REPO_URL
                        env.SOURCE_CRED_ID = env.GITLAB_CRED_ID
                        env.TARGET_CRED_ID = env.GITHUB_CRED_ID
                        env.SOURCE_NAME = 'GitLab'
                        env.TARGET_NAME = 'GitHub'
                    } else {
                        echo "Detected webhook from GitHub or manual trigger"
                    }
                    
                    // Get branch from webhook or use parameter
                    // Only use env.ref if it really is a Git ref (refs/heads/…)
                    if (env.ref?.startsWith('refs/heads/')) {
                        env.SYNC_BRANCH = env.ref.replaceFirst('refs/heads/', '')
                        echo "Branch from webhook: ${env.SYNC_BRANCH}"
                    } else {
                        env.SYNC_BRANCH = params.BRANCH_NAME
                        echo "Using configured branch: ${env.SYNC_BRANCH}"
                    }
                    
                    // Only proceed if it's the branch we care about
                    if (env.SYNC_BRANCH != params.BRANCH_NAME) {
                        echo "Webhook is for branch ${env.SYNC_BRANCH}, but we only sync ${params.BRANCH_NAME}. Aborting."
                        currentBuild.result = 'ABORTED'
                        error("Webhook for non-monitored branch")
                    }
                    
                    echo "Will sync from ${env.SOURCE_NAME} to ${env.TARGET_NAME}"
                }
            }
        }
        
        stage('Clone Source Repo') {
            steps {
                script {
                    // Clean workspace
                    deleteDir()
                    
                    echo "Cloning source repository from: ${env.SOURCE_REPO} (${env.SOURCE_NAME})"
                    
                    // Use different cloning methods based on the source platform
                    if (env.SOURCE_NAME == 'GitHub') {
                        withCredentials([string(credentialsId: env.SOURCE_CRED_ID, variable: 'GITHUB_TOKEN')]) {
                            // Create tokenized URL for GitHub
                            def tokenUrl = env.SOURCE_REPO.replace('https://', "https://x-access-token:${GITHUB_TOKEN}@")
                            sh """
                                git clone --single-branch --branch ${params.BRANCH_NAME} ${tokenUrl} ./repo
                            """
                        }
                    } else {
                        // Clone from GitLab using PAT
                        withCredentials([string(credentialsId: env.SOURCE_CRED_ID, variable: 'GITLAB_TOKEN')]) {
                            def tokenUrl = env.SOURCE_REPO.replace('https://', "https://oauth2:${GITLAB_TOKEN}@")
                            sh """
                                git clone --single-branch --branch ${params.BRANCH_NAME} ${tokenUrl} ./repo
                            """
                        }
                    }
                    
                    // Get commit info for logging
                    dir('repo') {
                        env.LATEST_COMMIT = sh(script: 'git log -1 --pretty=format:"%h - %an: %s"', returnStdout: true).trim()
                        echo "Latest commit: ${env.LATEST_COMMIT}"
                    }
                }
            }
        }
        
        stage('Configure Target Remote') {
            steps {
                script {
                    echo "Configuring target remote: ${env.TARGET_REPO} (${env.TARGET_NAME})"
                    
                    dir('repo') {
                        sh """
                            git remote add target ${env.TARGET_REPO}
                        """
                    }
                }
            }
        }
        
        stage('Fetch Target') {
            steps {
                script {
                    echo "Fetching from target to check for conflicts"
                    
                    dir('repo') {
                        // Fetch from target using appropriate credentials for the platform
                        if (env.TARGET_NAME == 'GitHub') {
                            withCredentials([string(credentialsId: env.TARGET_CRED_ID, variable: 'GITHUB_TOKEN')]) {
                                def tokenUrl = env.TARGET_REPO.replace('https://', "https://x-access-token:${GITHUB_TOKEN}@")
                                sh """
                                    git remote set-url target ${tokenUrl}
                                    git fetch target ${params.BRANCH_NAME}
                                """
                            }
                        } else {
                            withCredentials([string(credentialsId: env.TARGET_CRED_ID, variable: 'GITLAB_TOKEN')]) {
                                def tokenUrl = env.TARGET_REPO.replace('https://', "https://oauth2:${GITLAB_TOKEN}@")
                                sh """
                                    git remote set-url target ${tokenUrl}
                                    git fetch target ${params.BRANCH_NAME}
                                """
                            }
                        }
                        
                        // Check if target has commits we don't have
                        sh """
                            echo "Comparing branches:"
                            git log --oneline HEAD..target/${params.BRANCH_NAME} || echo "No unique commits on target"
                            git log --oneline target/${params.BRANCH_NAME}..HEAD || echo "No unique commits on source"
                        """
                    }
                }
            }
        }
        
        stage('Push to Target') {
            steps {
                script {
                    echo "Pushing to target repository ${env.TARGET_NAME}"
                    
                    dir('repo') {
                        try {
                            // Push using appropriate credentials for the platform
                            if (env.TARGET_NAME == 'GitHub') {
                                withCredentials([string(credentialsId: env.TARGET_CRED_ID, variable: 'GITHUB_TOKEN')]) {
                                    def tokenUrl = env.TARGET_REPO.replace('https://', "https://x-access-token:${GITHUB_TOKEN}@")
                                    sh """
                                        git remote set-url target ${tokenUrl}
                                        git push target ${params.BRANCH_NAME}:${params.BRANCH_NAME}
                                    """
                                }
                            } else {
                                withCredentials([string(credentialsId: env.TARGET_CRED_ID, variable: 'GITLAB_TOKEN')]) {
                                    def tokenUrl = env.TARGET_REPO.replace('https://', "https://oauth2:${GITLAB_TOKEN}@")
                                    sh """
                                        git remote set-url target ${tokenUrl}
                                        git push target ${params.BRANCH_NAME}:${params.BRANCH_NAME}
                                    """
                                }
                            }
                            
                            env.SYNC_SUCCESS = 'true'
                            env.SYNC_MESSAGE = "Successfully synchronized ${params.BRANCH_NAME} branch from ${env.SOURCE_NAME} to ${env.TARGET_NAME}"
                        } catch (Exception e) {
                            env.SYNC_SUCCESS = 'false'
                            env.SYNC_MESSAGE = "Failed to synchronize: ${e.getMessage()}"
                            
                            // Capture git status for better error reporting
                            sh """
                                echo "Git status:"
                                git status
                                echo "Git remote -v:"
                                git remote -v
                                echo "Last commits source:"
                                git log --oneline -n 5
                                echo "Last commits target:"
                                git log --oneline -n 5 target/${params.BRANCH_NAME} || echo "Cannot access target branch"
                            """
                            
                            echo "Error during push: ${e.getMessage()}"
                        }
                    }
                }
            }
        }
        
        stage('Report Status') {
            steps {
                script {
                    if (env.SYNC_SUCCESS == 'true') {
                        echo """
                        ✅ SYNCHRONIZATION SUCCESS
                        -------------------------
                        Direction: ${env.SOURCE_NAME} → ${env.TARGET_NAME}
                        Branch: ${params.BRANCH_NAME}
                        Latest commit: ${env.LATEST_COMMIT}
                        Message: ${env.SYNC_MESSAGE}
                        """
                    } else {
                        echo """
                        ❌ SYNCHRONIZATION FAILURE
                        -------------------------
                        Direction: ${env.SOURCE_NAME} → ${env.TARGET_NAME}
                        Branch: ${params.BRANCH_NAME}
                        Latest commit: ${env.LATEST_COMMIT}
                        Error: ${env.SYNC_MESSAGE}
                        
                        This typically happens when:
                        1. The target repository has commits that the source doesn't
                        2. Force push is disabled on the target repository
                        3. Authentication issues with the target repository
                        4. Branch protection rules on the target repository
                        
                        RESOLUTION OPTIONS:
                        1. Manually merge the branches
                        2. Force push (if appropriate)
                        3. Update the source repository to include target changes first
                        
                        Please check the Jenkins console log for more details.
                        """
                        
                        error "Repository synchronization failed. See logs for details."
                    }
                }
            }
        }
    }
    
    post {
        always {
            // No need to clean up git-credentials or ssh config - using in-memory token authentication
            cleanWs()
        }
        success {
            echo "Synchronization completed successfully"
        }
        failure {
            echo "Synchronization failed - check logs for details"
        }
    }
}