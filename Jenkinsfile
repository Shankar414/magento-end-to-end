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

        stage('Magento-default-Setup') {
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
                    // Remove existing containers by name
    sh '''
        set -e
        for c in "${PROJECT_NAME}_mysql" "${PROJECT_NAME}_redis" "${PROJECT_NAME}_elasticsearch" "${PROJECT_NAME}_phpfpm" "${PROJECT_NAME}_nginx"; do
          if docker ps -a --format '{{.Names}}' | grep -q "^${c}$"; then
            echo "Removing container ${c}"
            docker rm -f "${c}"
          else
            echo "Container ${c} does not exist"
          fi
        done
    '''

    // Now pull and start containers
    sh '''
        docker compose pull
        docker compose up -d
        docker compose exec php-fpm bash -lc 'php -v && composer --version'
        docker compose exec php-fpm bash -lc 'composer config --global http-basic.repo.magento.com 5be721829782cab5ab5f358283cce348 e3c319f62e2140eb6ac0998897de2244
'
        docker compose exec php-fpm bash -lc 'composer config --global repo.magento composer https://repo.magento.com'
        docker compose exec php-fpm bash -lc 'composer config --global allow-plugins true'
        docker compose exec php-fpm bash -lc 'rm -rf temp'
        docker compose exec php-fpm bash -lc 'composer create-project --repository-url=https://repo.magento.com/ magento/project-enterprise-edition=${MAGENTO_VERSION} temp'
        docker compose exec php-fpm bash -lc 'cp -a temp/. .'
        docker compose exec php-fpm bash -lc 'rm -rf temp'
        docker compose exec php-fpm bash -lc 'composer install --no-interaction --no-progress --no-suggest'
        docker compose exec php-fpm bash -lc 'chmod -R 770 /var/www/html'
        docker compose exec php-fpm bash -lc "
cd /var/www/html && \
php bin/magento setup:install \
 --base-url="https://${MAGENTO_HOST}/" \
 --db-host='mysql' \
 --db-name='${MAGENTO_DB_NAME}' \
 --db-user='${MAGENTO_DB_USER}' \
 --db-password='${MAGENTO_DB_PASSWORD}' \
 --admin-firstname=mage \
 --admin-lastname='admin' \
 --admin-email=example@gmail.com \
 --admin-user=mageadmin \
 --admin-password=changeme \
 --language='en_US' \
 --currency='USD' \
 --timezone='UTC' \
 --use-rewrites=1 \
 --search-engine='elasticsearch7' \
 --elasticsearch-host='elasticsearch' \
 --elasticsearch-port=9200 \
 --session-save=redis \
 --session-save-redis-host='redis' \
 --session-save-redis-port=6379 \
 --cache-backend=redis \
 --cache-backend-redis-server='redis' \
 --cache-backend-redis-port=6379 \
 --page-cache=redis \
 --page-cache-redis-server='redis' \
 --page-cache-redis-port=6379
"
    '''
                }
            }
        }
    }
}
