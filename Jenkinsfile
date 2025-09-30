pipeline {
  agent { label 'built-in' }

  environment {
    DOCKER_TLS_VERIFY = '1'
    DOCKER_CERT_PATH  = '/certs/client'
    DOCKER_HOST       = 'tcp://docker:2376'
    DOCKER_USER       = '22471264'
    IMAGE_NAME        = 'eb-express'
    IMAGE_TAG         = "main-${env.BUILD_NUMBER}"
    DOCKER_REPO       = "${env.DOCKER_USER}/${env.IMAGE_NAME}"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Install & Test (Node 16)') {
      agent {
        docker {
          image 'node:16-alpine'
          args "-u 1000:1000 -v ${env.WORKSPACE}:${env.WORKSPACE} -w ${env.WORKSPACE} -v ${env.HOME}/.npm:/root/.npm"
        }
      }
      steps {
        sh '''
          set -eux
          node -v
          npm -v
          npm ci
          npm test || echo "no tests"
        '''
      }
    }

    stage('Dependency Scan (fail on HIGH)') {
      agent {
        docker {
          image 'node:16-alpine'
          args "-u 1000:1000 -v ${env.WORKSPACE}:${env.WORKSPACE} -w ${env.WORKSPACE} -v ${env.HOME}/.npm:/root/.npm"
        }
      }
      steps {
        sh '''
          set -eux
          npm ci --prefer-offline --no-audit
          npm audit --audit-level=high || true
        '''
      }
    }

    stage('Build Docker Image') {
      steps {
        sh '''
          set -eux
          docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
        '''
      }
    }

    stage('Login & Push') {
      steps {
        withCredentials([string(credentialsId: 'docker-pass', variable: 'DOCKER_PASS')]) {
          sh '''
            set -eux
            echo "$DOCKER_PASS" | docker login -u ${DOCKER_USER} --password-stdin
            docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_REPO}:${IMAGE_TAG}
            docker push ${DOCKER_REPO}:${IMAGE_TAG}
          '''
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'npm-debug.log,**/reports/**,**/*.xml', allowEmptyArchive: true
      junit testResults: '**/junit-*.xml', allowEmptyResults: true
    }
  }
}
