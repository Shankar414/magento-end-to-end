pipeline {
    agent any

    options {
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }

    parameters {
        string(name: 'BRANCH_NAME', defaultValue: 'main', description: 'Branch to build')
    }

    environment {
        GIT_URL = 'https://github.com/Shankar414/magento-end-to-end.git' // define once for reuse
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Checking out branch: ${params.BRANCH_NAME}"
                git branch: "${params.BRANCH_NAME}", url: "${env.GIT_URL}"
            }
        }

        stage('Load Env') {
            steps {
                script {
                    def envVars = readFile('.env').trim()
                    envVars.split('\n').each { line ->
                        def pair = line.split('=')
                        if (pair.length == 2) {
                            env."${pair[0]}" = pair[1]
                        }
                    }
                }
                echo "PROJECT_NAME is ${env.PROJECT_NAME}"
            }
        }

        stage('Setup') {
            steps {
                script {
                    // Setup local users, groups, permissions
                    sh """
                        set -e
                        echo "setting up the local setup"
                        sudo groupadd "$PROJECT_NAME" || echo "group exists"
                        sudo useradd -m "$PROJECT_NAME" -g "$PROJECT_NAME" || echo "user exists"
                        sudo chmod 770 "/home/$PROJECT_NAME"
                        sudo usermod -aG "$PROJECT_NAME" jenkins
                        sudo usermod -aG "$PROJECT_NAME" ubuntu
                        sudo usermod -aG docker "$PROJECT_NAME"
                        sudo mkdir -p "/home/$PROJECT_NAME/$PROJECT_NAME-$PROJECT_ENVIRONMENT"
                        sudo chown -R jenkins:striff "/home/$PROJECT_NAME/$PROJECT_NAME-$PROJECT_ENVIRONMENT"
                        sudo chmod -R 770 "/home/$PROJECT_NAME/$PROJECT_NAME-$PROJECT_ENVIRONMENT"
                    """
                    // Mark safe directory for git
                    sh 'git config --global --add safe.directory /home/striff/striff-dev'
                }
                dir("/home/${env.PROJECT_NAME}/${env.PROJECT_NAME}-${env.PROJECT_ENVIRONMENT}") {
                    // Checkout git repo in target directory
                    git branch: "${params.BRANCH_NAME}", url: "${env.GIT_URL}"

                    // Bring up docker containers
                    sh '''
                        #!/bin/bash
                        docker compose down
                        docker compose pull
                        docker compose up -d
                    '''
                }
            }
        }
    }
}
