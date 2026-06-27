pipeline {
  agent any

  environment {
    REGISTRY = 'localhost:5000'
    GIT_CREDENTIALS_ID = 'github-creds'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Set image name') {
      steps {
        script {
          env.IMAGE_NAME = "${REGISTRY}/gitops-springboot:${env.BUILD_NUMBER}"
        }
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
        sh "sed -i 's|image: .*|image: ${IMAGE_NAME}|g' manifests/k8s/deployment.yaml"
      }
    }

    stage('Commit and push manifest change') {
      steps {
        withCredentials([usernamePassword(credentialsId: env.GIT_CREDENTIALS_ID, usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
          sh '''
            git config user.email "jenkins@example.com"
            git config user.name "jenkins-ci"
            git add manifests/k8s/deployment.yaml
            if git diff --cached --quiet; then
              echo "No manifest changes to commit"
            else
              git commit -m "Update GitOps image to ${IMAGE_NAME}"
              git remote set-url origin "https://${GIT_USERNAME}:${GIT_PASSWORD}@$(git remote get-url origin | sed -e 's#https://##')"
              git push origin HEAD:${GIT_BRANCH:-main}
            fi
          '''
        }
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
