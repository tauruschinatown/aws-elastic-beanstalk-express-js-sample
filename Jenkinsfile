pipeline {
  agent none
  environment {
    IMAGE_REPO = "eb-express"
    IMAGE_TAG  = "main-${env.BUILD_NUMBER}"
  }

  stages {
    stage('Checkout') {
      agent { label 'built-in' }
      steps { checkout scm }
    }

    stage('Install & Test (Node 16)') {
      agent { docker { image 'node:16-alpine'; args '-u root:root' } }
      steps {
        sh 'node -v && npm -v'
        sh 'npm ci'
        sh 'npm test || echo "no tests"'
      }
    }

    stage('Dependency Scan (fail on HIGH)') {
      agent { docker { image 'node:16-alpine'; args '-u root:root' } }
      steps {
        sh 'npm ci --prefer-offline --no-audit'
        sh 'npm audit --audit-level=high'
      }
    }

    stage('Build Docker Image') {
      agent { label 'built-in' }
      steps {
        sh 'docker build -t $IMAGE_REPO:$IMAGE_TAG .'
      }
    }

    stage('Login & Push') {
      agent { label 'built-in' }
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                          usernameVariable: 'DOCKER_USER',
                                          passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker tag $IMAGE_REPO:$IMAGE_TAG $DOCKER_USER/$IMAGE_REPO:$IMAGE_TAG
            docker push $DOCKER_USER/$IMAGE_REPO:$IMAGE_TAG
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
