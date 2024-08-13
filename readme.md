# Jenkins Pipeline for GitHub to Gitea Migration

This Jenkins pipeline automates the process of migrating repositories from GitHub to Gitea. It performs the following steps:

1. **Fetch Repositories from GitHub**: Retrieves a list of all repositories (including private ones) from GitHub.
2. **Clone and Push Repositories**: Clones each repository from GitHub to the Jenkins workspace and pushes it to Gitea.
3. **Create Repositories in Gitea**: Checks if the repository already exists in Gitea and creates it if it doesn't.
4. **Configure and Push**: Configures Git settings, adds Gitea as a remote, and pushes the repository to Gitea.

## Prerequisites

1. **Jenkins Setup**:
   - Jenkins instance with the ability to execute pipeline scripts.
   - Installed plugins: `Pipeline`, `Git`, and any additional plugins required for your specific setup.

2. **Credentials**:
   - **GitHub Token**: A personal access token with `repo` scope for accessing GitHub repositories.
   - **Gitea Token**: A personal access token with permissions to create and manage repositories in Gitea.

3. **Configuration**:
   - Replace placeholder values with actual information:
     - `GITEA_IP_XR_DOMAIN`: Your Gitea server's IP or domain.
     - `YXUR_GITEA_USERNAME`: Your Gitea username.
     - `YXUR_GUTHUB_USERNAME`: Your GitHub username.

## Pipeline Configuration

```groovy
pipeline {
    agent any

    environment {
        githubToken    = credentials('github-token')  // GitHub personal access token
        giteaToken     = credentials('gitea-token')   // Gitea personal access token
        giteaServer    = 'GITEA_IP_XR_DOMAIN'         // Replace with your Gitea server's IP/domain
        giteaUsername  = 'YXUR_GITEA_USERNAME'        // Replace with your Gitea username
        githubUsername = 'YXUR_GUTHUB_USERNAME'       // Replace with your GitHub username
    }

    options {
        skipDefaultCheckout(true)  // Skip the default checkout step
    }

    stages {
        stage('Fetch Repository List') {
            steps {
                script {
                    // Fetch all repositories (including private) from GitHub
                    def repoList = sh(returnStdout: true, script: """
                        curl -s -H 'Authorization: token ${githubToken}' https://api.github.com/user/repos?visibility=all
                    """).trim()

                    // Decode repository list and prepare for processing
                    repos = sh(returnStdout: true, script: """
                        echo '${repoList}' | jq -r '.[] | {name: .name, description: .description, private: .private} | @base64'
                    """).split('\n')

                    echo "Repositories to clone and push: ${repos.join(', ')}"
                }
            }
        }

        stage('Clone and Push Repositories') {
            steps {
                script {
                    repos.each { repoData ->
                        // Decode base64-encoded repository data
                        def decodedRepoData = sh(returnStdout: true, script: """
                            echo ${repoData} | base64 --decode | jq -r 'to_entries | map("\\(.key)=\\(.value | tostring)") | .[]'
                        """).trim()
                        def repoName = decodedRepoData.split("\n").find { it.startsWith('name=') }.split('=')[1].trim()
                        def repoDescription = decodedRepoData.split("\n").find { it.startsWith('description=') }.split('=')[1].trim()
                        def repoPrivate = decodedRepoData.split("\n").find { it.startsWith('private=') }.split('=')[1].trim()

                        stage("Processing ${repoName}") {
                            dir("${env.WORKSPACE}/${repoName}") {
                                // Clone the repository if not already cloned
                                sh """
                                    if [ ! -d .git ]; then
                                        git clone https://${githubToken}@github.com/${githubUsername}/${repoName}.git .
                                    else
                                        echo 'Repository already cloned'
                                    fi
                                """

                                // Check if the repository exists in Gitea
                                def repoExists = sh(returnStatus: true, script: """
                                    curl -s -H 'Authorization: token ${giteaToken}' https://${giteaServer}/api/v1/user/repos/${repoName}
                                """)

                                // Create the repository in Gitea if it doesn't exist
                                if (repoExists != 0) {
                                    sh """
                                        curl -k -X POST https://${giteaServer}/api/v1/user/repos \
                                        -H 'Authorization: token ${giteaToken}' \
                                        -H 'Content-Type: application/json' \
                                        -d '{"name": "${repoName}", "private": ${repoPrivate}, "description": "${repoDescription}"}'
                                    """
                                } else {
                                    echo "Repository ${repoName} already exists in Gitea."
                                }

                                // Increase Git buffer size and disable SSL verification
                                sh "git config http.postBuffer 524288000"
                                sh "git config http.sslVerify false"

                                // Add Gitea as a remote and push the repository
                                sh """
                                    git remote add gitea https://${giteaToken}@${giteaServer}/${giteaUsername}/${repoName}.git
                                    git push -u gitea master
                                """
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            // Clean up workspace after build
            cleanWs(cleanWhenNotBuilt: false,
                    deleteDirs: true,
                    disableDeferredWipeout: true,
                    notFailBuild: true)
        }
    }
}

---