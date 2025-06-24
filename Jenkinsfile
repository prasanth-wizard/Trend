pipeline {
  agent any

  environment {
    DOCKER_IMAGE = "prasanth0003/react_app"
    DOCKER_CREDENTIALS_ID = 'dockerhub'
  }

  stages {
    stage('Clone Repository') {
      steps {
        git branch: 'main', credentialsId: 'github', url: 'https://github.com/prasanth-wizard/Trend.git'
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          docker.build("${DOCKER_IMAGE}")
        }
      }
    }

    stage('Login & Push to DockerHub') {
      steps {
        script {
          docker.withRegistry('', DOCKER_CREDENTIALS_ID) {
            docker.image("${DOCKER_IMAGE}").push("latest")
          }
        }
      }
    }

    stage('Deploy to EKS') {
      steps {
        sh 'kubectl apply -f deployment.yaml'
        sh 'kubectl apply -f service.yaml'
        sh 'kubectl get svc'
      }
    }
  }
}

