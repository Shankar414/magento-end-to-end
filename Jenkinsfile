pipeline {
    agent any

    options {
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }

    parameters {
        string(name: 'BRANCH_NAME', defaultValue: 'main', description: 'Branch to build')
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Checking out branch: ${params.BRANCH_NAME}"
                git branch: "${params.BRANCH_NAME}", url: 'https://github.com/Shankar414/magento-end-to-end.git'
            }
        }

      stages {
        stage('Load Env') {
            steps {
                script {
                    def envVars = readFile('.env').trim()
                    envVars.split("\n").each { line ->
                        def pair = line.split('=')
                        if (pair.length == 2) {
                            // export environment variables dynamically
                            env."${pair[0]}" = pair[1]
                        }
                    }
                }
                echo "PROJECT_NAME is ${env.PROJECT_NAME}"
            }
        }

      stage('Setup') {
            steps {
                sh '''
                echo "setting up the local setup"
                sudo groupadd "$PROJECT_NAME" || echo "group exists"
                sudo useradd -m "$PROJECT_NAME" -g "$PROJECT_NAME" || echo "user exists"
                sudo chmod 770 "/home/$PROJECT_NAME"
                sudo chmod 770 "/home/$PROJECT_NAME"
                sudo usermod -aG "$PROJECT_NAME" jenkins
                sudo usermod -aG "$PROJECT_NAME" ubuntu
                sudo usermod -aG docker "$PROJECT_NAME"
                sudo mkdir -p "/home/$PROJECT_NAME/$PROJECT_NAME-$PROJECT_ENVIRONMENT"

                '''
            }
        }
    }
}

}