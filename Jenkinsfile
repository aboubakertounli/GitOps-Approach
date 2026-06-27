pipeline {
  agent any

  environment {
    REGISTRY = 'localhost:5000'
    IMAGE_NAME = "${REGISTRY}/gitops-springboot:latest"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build') {
      steps {
        sh 'mvn -B -DskipTests package'
      }
    }

    stage('Unit tests') {
      steps {
        sh 'mvn test'
      }
    }

    stage('Build Docker image') {
      steps {
        sh 'docker build -t ${IMAGE_NAME} .'
      }
    }

    stage('Push image to local registry') {
      steps {
        sh 'docker push ${IMAGE_NAME}'
      }
    }

    stage('Update GitOps manifest') {
      steps {
        sh "sed -i 's|ghcr.io/OWNER/REPO/spring-boot-gitops:latest|${IMAGE_NAME}|g' manifests/k8s/deployment.yaml"
      }
    }
  }

  post {
    success {
      echo "Jenkins build finished successfully. Image pushed to ${IMAGE_NAME}."
    }
    failure {
      echo "Jenkins pipeline failed.";
    }
  }
}
