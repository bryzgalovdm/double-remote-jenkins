// Boilerplate code generated by claude. So far here as a placeholder

pipeline {
    agent any

    triggers {
        githubPush()
    }

    stages {
        stage('Sync Repositories') {
            steps {
                script {
                    // Clone from GitHub
                    sh """
                        git clone https://github.com/your-org/your-repo.git
                        cd your-repo

                        # Add GitLab remote if not exists
                        git remote -v | grep gitlab || git remote add gitlab https://gitlab.com/your-org/your-repo.git

                        # Push to GitLab
                        git push gitlab --all
                    """
                }
            }
        }
    }
}