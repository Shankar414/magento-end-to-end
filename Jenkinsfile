pipeline {
    agent any

    environment {
        APP_ENV = 'dev'
        NODE_VERSION = '20'
    }

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

      stage('Setup') {
            steps {
                sh '''
                echo "setting up the local setup"
                sudo groupadd $PROJECT_NAME || echo 'group exists'
                '''
            }
        }
    }
}