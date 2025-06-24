pipeline {
  agent any

  environment {
    DOCKER_IMAGE = "prasanth0003/react_app:latest"
    DOCKER_CREDENTIALS_ID = 'dockerhub'
    GIT_BRANCH = 'main'
  }

  stages {
    stage('Clone Repository') {
      steps {
        // ✅ Explicitly specify the branch
        git branch: "${GIT_BRANCH}", credentialsId: 'github', url: 'https://github.com/prasanth-wizard/Trend.git'
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          docker.build("react_app")
        }
      }
    }

    stage('Login & Push to DockerHub') {
      steps {
        script {
          docker.withRegistry('', "${DOCKER_CREDENTIALS_ID}") {
            docker.image("react_app").push("latest")
          }
        }
      }
    }

    stage('Deploy to EKS') {
      steps {
        // ✅ Apply your Kubernetes manifests
        sh 'kubectl apply -f deployment.yaml'
        sh 'kubectl apply -f service.yaml'

        // Optional: View services
        sh 'kubectl get svc'
      }
    }
  }
}

