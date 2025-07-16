
# React E-Commerce App – CI/CD Deployment using Terraform, Jenkins, and AWS EKS

This project demonstrates my ability to provision cloud infrastructure with Terraform, containerize and deploy a production-ready ReactJS application on AWS EKS, and automate the full CI/CD pipeline using Jenkins. It also includes monitoring setup using Prometheus and Grafana via Helm.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Technology Stack](#technology-stack)
3. [Workflow Summary](#workflow-summary)
4. [Infrastructure Provisioning with Terraform](#infrastructure-provisioning-with-terraform)
5. [Dockerization](#dockerization)
6. [CI/CD Pipeline with Jenkins](#cicd-pipeline-with-jenkins)
7. [EKS Deployment](#eks-deployment)
8. [Monitoring Setup (via Helm)](#monitoring-setup-via-helm)
9. [Project Outcome](#project-outcome)
10. [Contact](#contact)

---

## Project Overview

- Provision AWS infrastructure (VPC, EC2, EKS) using Terraform
- Dockerize a ReactJS application and push images to DockerHub
- Configure Jenkins for automated CI/CD with GitHub integration
- Deploy the app to AWS EKS via kubectl from Jenkins
- Set up monitoring system using Prometheus and Grafana via Helm

---

## Technology Stack

| Category           | Tools / Services                             |
|--------------------|----------------------------------------------|
| Frontend           | ReactJS                                      |
| Infrastructure     | Terraform (VPC, EC2, EKS)                     |
| Containerization   | Docker                                       |
| Image Registry     | DockerHub                                    |
| CI/CD              | Jenkins (Declarative Pipeline)               |
| Source Control     | GitHub + GitHub Webhook                      |
| Orchestration      | Kubernetes (AWS EKS)                         |
| Monitoring         | Prometheus, Grafana (via Helm)               |
| OS / Platform      | AWS EC2, Ubuntu, Amazon Linux                |

---

## Workflow Summary

1. Clone source code from GitHub and push to personal repo
2. Use Terraform to provision AWS infrastructure:
   - VPC, subnets, internet gateway
   - EC2 instance for Jenkins
   - EKS cluster and node group
3. Dockerize the app and push to DockerHub
4. Configure Jenkins to:
   - Build and push Docker image
   - Deploy app to EKS via kubectl
5. Monitor the app and cluster using Prometheus and Grafana (Helm)

---

## Infrastructure Provisioning with Terraform

### main.tf

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "5.45.0"
    }
  }
  required_version = ">= 1.5.0"
}

provider "aws" {
  region = var.aws_region
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {
    Name = "trend-vpc"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "trend-igw"
  }
}

# Public Subnets
resource "aws_subnet" "public_subnet_1" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true
  tags = {
    Name = "trend-public-subnet-1"
  }
}

resource "aws_subnet" "public_subnet_2" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.2.0/24"
  availability_zone       = "us-east-1b"
  map_public_ip_on_launch = true
  tags = {
    Name = "trend-public-subnet-2"
  }
}

# Route Table
resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = {
    Name = "trend-public-rt"
  }
}

# Route Table Association
resource "aws_route_table_association" "a1" {
  subnet_id      = aws_subnet.public_subnet_1.id
  route_table_id = aws_route_table.public_rt.id
}

resource "aws_route_table_association" "a2" {
  subnet_id      = aws_subnet.public_subnet_2.id
  route_table_id = aws_route_table.public_rt.id
}

# Security Group
resource "aws_security_group" "jenkins_sg" {
  name        = "jenkins-sg"
  description = "Allow ports for Jenkins and app"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 3000
    to_port     = 3000
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "jenkins-sg"
  }
}

# IAM Role for EC2
resource "aws_iam_role" "ec2_role" {
  name = "trend-ec2-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
  })
}

# Attach policies to the IAM Role
resource "aws_iam_role_policy_attachment" "attach_all" {
  for_each = toset([
    "arn:aws:iam::aws:policy/AdministratorAccess",
    "arn:aws:iam::aws:policy/AmazonEC2FullAccess",
    "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy",
    "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy",
    "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy",
    "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryFullAccess",
    "arn:aws:iam::aws:policy/AWSCodeBuildAdminAccess",
    "arn:aws:iam::aws:policy/CloudWatchFullAccess",
    "arn:aws:iam::aws:policy/IAMFullAccess"
  ])
  role       = aws_iam_role.ec2_role.name
  policy_arn = each.value
}

# IAM Instance Profile
resource "aws_iam_instance_profile" "ec2_profile" {
  name = "trend-ec2-profile"
  role = aws_iam_role.ec2_role.name
}

# EC2 Instance
resource "aws_instance" "jenkins_ec2" {
  ami                    = "ami-053b0d53c279acc90" # Ubuntu 22.04 in us-east-1
  instance_type          = "t2.medium"
  subnet_id              = aws_subnet.public_subnet_1.id
  key_name               = "keypair"
  vpc_security_group_ids = [aws_security_group.jenkins_sg.id]
  iam_instance_profile   = aws_iam_instance_profile.ec2_profile.name

  user_data = <<-EOF
              #!/bin/bash
              sudo apt update -y
              sudo apt install openjdk-17-jdk -y
              sudo apt install docker.io -y
              sudo systemctl enable docker
              sudo usermod -aG docker ubuntu
              sudo apt install unzip -y
              curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
                /usr/share/keyrings/jenkins-keyring.asc > /dev/null
              echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
                https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
                /etc/apt/sources.list.d/jenkins.list > /dev/null
              sudo apt update -y
              sudo apt install jenkins -y
              sudo systemctl start jenkins
              sudo systemctl enable jenkins
              sudo apt install -y curl
              curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
              sudo chmod +x kubectl
              sudo mv kubectl /usr/local/bin/
            EOF

  tags = {
    Name = "trend-jenkins-ec2"
  }
}

Outputs.tf

output "ec2_instance_public_ip" {
  description = "Public IP of the Jenkins EC2 instance"
  value       = aws_instance.jenkins_ec2.public_ip
}

output "vpc_id" {
  description = "ID of the created VPC"
  value       = aws_vpc.main.id
}

output "iam_instance_profile_name" {
  description = "Name of the IAM instance profile"
  value       = aws_iam_instance_profile.ec2_profile.name
}

variables.tf

variable "aws_region" {
  default = "us-east-1"
}

````
```Terraform Execution:

terraform init
terraform validate
terraform plan
terraform apply

```

---
### Commands

```bash
terraform init
terraform apply -auto-approve
```

---

## Dockerization

### Dockerfile

```dockerfile
FROM nginx:alpine
WORKDIR /usr/share/nginx/html
RUN rm -rf ./*
COPY build/ .
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### .dockerignore

```
node_modules
.git
Dockerfile
.gitignore
README.md
```

### Build & Push

```bash
docker build -t prasanth0003/terraform-react-app .
docker push prasanth0003/terraform-react-app:latest
```

---

## CI/CD Pipeline with Jenkins

### Jenkins Setup

* Installed plugins: Docker, Docker Pipeline, Git, Kubernetes CLI
* Configured GitHub webhook for `main` branch
* Used SSH key-based login from Jenkins EC2 to EKS nodes
* Credentials stored securely in Jenkins (GitHub, DockerHub, SSH)

### Jenkinsfile (Declarative Pipeline)

```groovy
pipeline {
    agent any

    environment {
        IMAGE = "prasanth0003/terraform-react-app"
    }

    stages {
        stage('Clone Repo') {
            steps {
                git branch: 'main', url: 'https://github.com/prasanth-wizard/Trend.git'
            }
        }

        stage('Install & Build') {
            steps {
                sh 'npm install'
                sh 'npm run build'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    dockerImage = docker.build("${IMAGE}:${BUILD_NUMBER}")
                }

                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    sh "docker push ${IMAGE}:${BUILD_NUMBER}"
                    sh "docker tag ${IMAGE}:${BUILD_NUMBER} ${IMAGE}:latest"
                    sh "docker push ${IMAGE}:latest"
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                aws eks update-kubeconfig --region us-east-1 --name terraform-eks-cluster
                kubectl apply -f deployment.yaml
                kubectl apply -f service.yaml
                '''
            }
        }
    }

    post {
        success {
            echo 'Deployment Successful'
        }
        failure {
            echo 'Deployment Failed'
        }
    }
}
```

---

## EKS Deployment

### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecommerce-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ecommerce
  template:
    metadata:
      labels:
        app: ecommerce
    spec:
      containers:
      - name: ecommerce
        image: prasanth0003/terraform-react-app:latest
        ports:
        - containerPort: 80
```

### service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ecommerce-service
spec:
  type: NodePort
  selector:
    app: ecommerce
  ports:
    - port: 3000
      targetPort: 80
      nodePort: 31000
```

---

## Monitoring Setup (via Helm)

* Monitoring stack deployed using **Helm charts** inside EKS
* Tools used:

  * **Prometheus**: For collecting and storing metrics
  * **Grafana**: For visualizing application and cluster metrics
* Configured Prometheus to use Kubernetes service discovery
* Accessed Grafana via NodePort service in the cluster

### Helm Commands Used

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack
helm install grafana grafana/grafana
```

* Logged into Grafana and connected to Prometheus as the data source to build dashboards.

---

## Project Outcome

* Provisioned AWS infrastructure with **Terraform**
* Dockerized and pushed React app image to **DockerHub**
* Configured **Jenkins** CI/CD pipeline for automated deployment
* Deployed application to **AWS EKS** using `kubectl` from Jenkins
* Deployed **Prometheus and Grafana using Helm** to monitor the EKS cluster
* Successfully automated code → container → cloud → monitoring pipeline

---

## Contact

* GitHub: [https://github.com/prasanth-wizard](https://github.com/prasanth-wizard)
* DockerHub: [https://hub.docker.com/u/prasanth0003](https://hub.docker.com/u/prasanth0003)

```
