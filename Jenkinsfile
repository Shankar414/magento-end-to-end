pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    disableConcurrentBuilds()
    timeout(time: 60, unit: 'MINUTES')
  }

  parameters {
    string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Branch to build/deploy')
    booleanParam(name: 'DOCKER_LOGIN', defaultValue: false, description: 'Login to private registry before pulling images')
    credentials(name: 'DOCKER_REGISTRY_CREDENTIALS', defaultValue: '', description: 'Credentials ID for private registry (username/password)', required: false)

    string(name: 'PROJECT_NAME', defaultValue: 'magento', description: 'Used in Traefik labels and docker compose project name')
    string(name: 'MAGENTO_HOST', defaultValue: 'shop.dcw.dev', description: 'Public hostname routed by Traefik')
    choice(name: 'MAGENTO_MODE', choices: ['production', 'developer'], description: 'Magento mode')

    // Container names (must match your .env or override here)
    string(name: 'NGINX_CONTAINER_NAME', defaultValue: 'magento-nginx', description: 'nginx container name')
    string(name: 'FPM_CONTAINER_NAME', defaultValue: 'magento-fpm', description: 'php-fpm container name')
    string(name: 'MYSQL_CONTAINER_NAME', defaultValue: 'magento-mysql', description: 'mariadb/mysql container name')
    string(name: 'ELASTICSEARCH_CONTAINER_NAME', defaultValue: 'magento-elasticsearch', description: 'elasticsearch container name')
    string(name: 'REDIS_CONTAINER_NAME', defaultValue: 'magento-redis', description: 'redis container name')

    // Optional: override images at deploy-time
    string(name: 'NGINX_IMAGE', defaultValue: '', description: 'If set, overrides .env value')
    string(name: 'FPM_IMAGE', defaultValue: '', description: 'If set, overrides .env value')
    string(name: 'MYSQL_IMAGE', defaultValue: '', description: 'If set, overrides .env value')
    string(name: 'ELASTICSEARCH_IMAGE', defaultValue: '', description: 'If set, overrides .env value')
    string(name: 'REDIS_IMAGE', defaultValue: '', description: 'If set, overrides .env value')

    // Secrets - reference Jenkins credentials or leave empty to rely on .env
    password(name: 'MAGENTO_DB_PASSWORD', defaultValue: '', description: 'DB user password (optional override)')
    password(name: 'MAGENTO_DB_ROOT_PASSWORD', defaultValue: '', description: 'DB root password (optional override)')
    password(name: 'MAGENTO_REDIS_PASSWORD', defaultValue: '', description: 'Redis password (optional override)')
  }

  environment {
    REPO_DIR = "${env.WORKSPACE}"
    MAGENTO_DIR = "infra/traefik/magento"
    // Compose will use this as project name for network/volume scoping if needed
    COMPOSE_PROJECT_NAME = "${params.PROJECT_NAME}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout([$class: 'GitSCM', branches: [[name: params.GIT_BRANCH]], userRemoteConfigs: scm.userRemoteConfigs])
        sh 'git rev-parse --short HEAD'
      }
    }

    stage('Prepare .env') {
      steps {
        dir("${env.MAGENTO_DIR}") {
          script {
            // Create .env from example if missing
            sh '''#!/usr/bin/env bash
set -euo pipefail
if [ ! -f .env ]; then
  cp -v .env.example .env
fi
# Apply pipeline overrides into .env idempotently
apply_env() {
  local key="$1"; local val="$2"
  [ -z "$val" ] && return 0
  if grep -qE "^${key}=" .env; then
    sed -i "s|^${key}=.*|${key}=${val}|" .env
  else
    echo "${key}=${val}" >> .env
  fi
}
apply_env PROJECT_NAME '${PROJECT_NAME}'
apply_env MAGENTO_HOST '${MAGENTO_HOST}'
apply_env MAGENTO_MODE '${MAGENTO_MODE}'
apply_env NGINX_CONTAINER_NAME '${NGINX_CONTAINER_NAME}'
apply_env FPM_CONTAINER_NAME '${FPM_CONTAINER_NAME}'
apply_env MYSQL_CONTAINER_NAME '${MYSQL_CONTAINER_NAME}'
apply_env ELASTICSEARCH_CONTAINER_NAME '${ELASTICSEARCH_CONTAINER_NAME}'
apply_env REDIS_CONTAINER_NAME '${REDIS_CONTAINER_NAME}'
# Optional image overrides
apply_env NGINX_IMAGE '${NGINX_IMAGE}'
apply_env FPM_IMAGE '${FPM_IMAGE}'
apply_env MYSQL_IMAGE '${MYSQL_IMAGE}'
apply_env ELASTICSEARCH_IMAGE '${ELASTICSEARCH_IMAGE}'
apply_env REDIS_IMAGE '${REDIS_IMAGE}'
# Optional secret overrides
apply_env MAGENTO_DB_PASSWORD '${MAGENTO_DB_PASSWORD}'
apply_env MAGENTO_DB_ROOT_PASSWORD '${MAGENTO_DB_ROOT_PASSWORD}'
apply_env MAGENTO_REDIS_PASSWORD '${MAGENTO_REDIS_PASSWORD}'
'''
          }
        }
      }
    }

    stage('Docker Login (optional)') {
      when { expression { return params.DOCKER_LOGIN && params.DOCKER_REGISTRY_CREDENTIALS?.trim() } }
      steps {
        withCredentials([usernamePassword(credentialsId: params.DOCKER_REGISTRY_CREDENTIALS, passwordVariable: 'REG_PWD', usernameVariable: 'REG_USER')]) {
          sh 'echo "$REG_PWD" | docker login --username "$REG_USER" --password-stdin'
        }
      }
    }

    stage('Pull & Start') {
      steps {
        dir("${env.MAGENTO_DIR}") {
          sh '''#!/usr/bin/env bash
set -euo pipefail
# Show effective .env for debugging (redact secrets)
awk -F= '{ if ($1 ~ /PASSWORD|TOKEN|KEY|SECRET/) print $1"=****"; else print }' .env
# Ensure external proxy network exists (ignore error if already exists)
docker network inspect proxy >/dev/null 2>&1 || true
# Pull and start
DockerCmd="docker compose --env-file .env"
$DockerCmd pull || true
$DockerCmd up -d --remove-orphans
$DockerCmd ps
'''
        }
      }
    }

    stage('Warm up') {
      steps {
        // Give containers time to initialize services
        sleep time: 30, unit: 'SECONDS'
      }
    }

    stage('Magento Maintenance') {
      steps {
        dir("${env.MAGENTO_DIR}") {
          script {
            def fpm = params.FPM_CONTAINER_NAME
            sh """
#!/usr/bin/env bash
set -euo pipefail
run() { docker exec -i ${fpm} bash -lc "cd /var/www/html && \$*"; }
# Detect if Magento is installed (presence of app/etc/env.php)
if docker exec -i ${fpm} bash -lc 'test -f /var/www/html/app/etc/env.php'; then
  echo 'Magento appears installed. Running upgrade flow.'
  run php bin/magento maintenance:enable || true
  run php bin/magento setup:upgrade
else
  echo 'Fresh install detected. You may need to run setup:install manually with proper flags.'
  echo 'Skipping setup:install to avoid destructive operations.'
fi
# Common compile/deploy steps
run php bin/magento deploy:mode:set ${MAGENTO_MODE} -s || true
run php bin/magento setup:di:compile
run php bin/magento setup:static-content:deploy -f
run php bin/magento config:set web/unsecure/base_url http://${MAGENTO_HOST}/ || true
run php bin/magento config:set web/secure/base_url https://${MAGENTO_HOST}/ || true
run php bin/magento config:set web/secure/use_in_frontend 1 || true
run php bin/magento config:set web/secure/use_in_adminhtml 1 || true
run php bin/magento indexer:reindex || true
run php bin/magento cache:flush || true
run php bin/magento maintenance:disable || true
            """
          }
        }
      }
    }
  }

  post {
    success {
      echo 'Deployment completed successfully.'
    }
    failure {
      echo 'Deployment failed. Check console output for details.'
    }
    always {
      dir("${env.MAGENTO_DIR}") {
        sh 'docker compose --env-file .env ps || true'
      }
    }
  }
}

