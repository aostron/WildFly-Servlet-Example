pipeline {
    agent {
        label 'ubuntu-slave'
    }

    tools {
        maven 'M3' // Ensure Maven is configured in Jenkins global tool configuration as 'M3'
    }

    environment {
        GITHUB_USER = 'aostron'
        GITHUB_API_URL = 'https://api.github.com'
        ORIGINAL_OWNER = 'XadmaX'
        ORIGINAL_REPO = 'WildFly-Servlet-Example'
        FORKED_REPO = 'WildFly-Servlet-Example'
        NEW_BRANCH = 'test-branch'
        BASE_BRANCH = 'main'
    }

    stages {
        stage('Authenticate GitHub CLI') {
            steps {
                script {
                    def githubToken = withCredentials([string(credentialsId: 'full-access', variable: 'GITHUB_TOKEN')]) {
                        return env.GITHUB_TOKEN
                    }
                    
                    echo "Authenticating with GitHub CLI..."
                    sh """
                        echo "${githubToken}" | gh auth login --with-token
                    """
                }
            }
        }

        stage('Fork Repository') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'full-access', variable: 'GITHUB_TOKEN')]) {
                        echo "Forking the repository..."
                        sh """
                            gh repo fork ${ORIGINAL_OWNER}/${ORIGINAL_REPO} --clone=false --remote
                        """
                    }
                }
            }
        }

        stage('Clone Forked Repository and Create New Branch') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'full-access', variable: 'GITHUB_TOKEN')]) {
                        sh "git config --global credential.helper '!f() { sleep 1; echo username=${GITHUB_USER}; echo password=${GITHUB_TOKEN}; }; f'"
                        
                        def repositoryDir = "${WORKSPACE}/${FORKED_REPO}"
                        def cloneCommand = "git clone https://github.com/${GITHUB_USER}/${FORKED_REPO}.git '${repositoryDir}'"
                        
                        if (!fileExists(repositoryDir)) {
                            echo "Cloning the repository..."
                            sh cloneCommand
                        } else {
                            echo "Repository directory already exists. Fetching latest changes..."
                            dir(repositoryDir) {
                                sh """
                                    git fetch origin
                                    git checkout ${BASE_BRANCH}
                                    git pull origin ${BASE_BRANCH}
                                """
                            }
                        }

                        dir(repositoryDir) {
                            echo "Creating and pushing a new branch..."
                            def checkoutResult = sh(returnStatus: true, script: "git checkout -b ${NEW_BRANCH}")
                            if (checkoutResult != 0) {
                                echo "Switching to the existing branch '${NEW_BRANCH}'..."
                                sh """
                                    git checkout ${NEW_BRANCH}
                                    git pull origin ${NEW_BRANCH}
                                """
                            } else {
                                sh "git push --force origin ${NEW_BRANCH}"
                            }
                        }
                    }
                }
            }
        }

        stage('Build') {
            steps {
                cleanWs()
                git branch: 'main', url: "https://github.com/${GITHUB_USER}/${ORIGINAL_REPO}.git"
                sh "mvn clean package"
            }

            post {
                success {
                    archiveArtifacts artifacts: 'target/*.war', followSymlinks: false
                    stash name: 'warFile', includes: 'target/*.war'
                }
            }
        }

        stage('Deploy to WildFly') {
            steps {
                script {
                    def warFilePath = ''
                    def agents = ['ubuntu-slave']
                    def found = false

                    // Try to unstash the file from each agent
                    for (int i = 0; i < agents.size(); i++) {
                        if (found) {
                            break
                        }

                        node(agents[i]) {
                            try {
                                unstash 'warFile'
                                warFilePath = "${WORKSPACE}/target/devops-1.0-SNAPSHOT.war"
                                if (fileExists(warFilePath)) {
                                    echo "Found WAR file on ${agents[i]}: ${warFilePath}"
                                    found = true
                                }
                            } catch (Exception e) {
                                echo "WAR file not found on ${agents[i]}"
                            }
                        }
                    }

                    if (!found) {
                        error "WAR file not found on any agent."
                    }

                    // Debug: Print the final WAR file path
                    echo "WAR file path to deploy: ${warFilePath}"

                    // Deploy the WAR file
                    sshPublisher(publishers: [
                        sshPublisherDesc(
                            configName: 'WildFlyServer', // Name of the SSH server configuration
                            transfers: [
                                sshTransfer(
                                    sourceFiles: 'target/devops-1.0-SNAPSHOT.war', // Only specify the file name
                                    remoteDirectory: '',
                                    removePrefix: 'target', // This removes 'target' from the path
                                    execCommand: '', 
                                    execTimeout: 120000, 
                                    flatten: false, 
                                    makeEmptyDirs: false, 
                                    noDefaultExcludes: false, 
                                    patternSeparator: '[, ]+', 
                                    remoteDirectorySDF: false
                                )
                            ]
                        )
                    ])
                }
            }
        }
    }
}
