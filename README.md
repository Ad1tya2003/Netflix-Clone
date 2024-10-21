DevSecOps-Project

This repository contains a sample Netflix-style application that demonstrates the implementation of DevSecOps practices using Docker, Jenkins, SonarQube, Trivy, Kubernetes, Prometheus, Grafana, and ArgoCD.
Project Overview

In this project, we will:

    Set up and deploy the application using Docker on an EC2 instance.
    Implement security checks using SonarQube and Trivy.
    Configure a CI/CD pipeline using Jenkins.
    Deploy and monitor the app using Prometheus and Grafana.
    Deploy the app to a Kubernetes cluster with ArgoCD.

Phases and Steps
Phase 1: Initial Setup and Deployment
1. Launch EC2 Instance (Ubuntu 22.04)

    Provision an EC2 instance using Ubuntu 22.04.
    SSH into the instance.

2. Clone the Code

bash

sudo apt-get update
git clone https://github.com/N4si/DevSecOps-Project.git

3. Install Docker and Run the App Container

    Install Docker:

```bash

sudo apt-get install docker.io -y
sudo usermod -aG docker $USER
newgrp docker
sudo chmod 777 /var/run/docker.sock
```

    Build and run the Docker container for the app:

```bash

docker build -t netflix .
docker run -d --name netflix -p 8081:80 netflix:latest

4. Get an API Key for TMDB

    Sign up for an API key from TMDB.
    Build the Docker image again, this time using the API key:
```

bash

docker build --build-arg TMDB_V3_API_KEY=<your-api-key> -t netflix .

Phase 2: Security
1. Install SonarQube and Trivy

    Install SonarQube for code quality analysis:

bash

docker run -d --name sonar -p 9000:9000 sonarqube:lts-community

    Install Trivy for Docker image vulnerability scanning:

```bash

sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```

    Scan Docker images with Trivy:

bash

trivy image <imageid>

Phase 3: CI/CD Setup
1. Install Jenkins

    Install Jenkins and its dependencies:

```bash

sudo apt-get update
sudo apt-get install fontconfig openjdk-17-jre
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

    Access Jenkins at http://<your-server-ip>:8080.

2. Install Plugins in Jenkins

    Install plugins such as SonarQube Scanner, NodeJS, Docker, and OWASP Dependency-Check from Jenkins UI (Manage Jenkins > Plugins).

    Add credentials (DockerHub, SonarQube, etc.) under Manage Jenkins > Credentials.

3. Configure Pipeline in Jenkins

Add the following pipeline script in Jenkins:

```groovy

pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Clean Workspace') { steps { cleanWs() } }
        stage('Checkout from Git') {
            steps { git branch: 'main', url: 'https://github.com/N4si/DevSecOps-Project.git' }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix'''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script { waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' }
            }
        }
        stage('Install Dependencies') { steps { sh 'npm install' } }
        stage('OWASP Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Trivy FS Scan') { steps { sh 'trivy fs . > trivyfs.txt' } }
        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh 'docker build --build-arg TMDB_V3_API_KEY=<yourapikey> -t netflix .'
                        sh 'docker tag netflix dockerhub-user/netflix:latest'
                        sh 'docker push dockerhub-user/netflix:latest'
                    }
                }
            }
        }
        stage('Trivy Image Scan') { steps { sh 'trivy image dockerhub-user/netflix:latest > trivyimage.txt' } }
        stage('Deploy to Container') { steps { sh 'docker run -d --name netflix -p 8081:80 dockerhub-user/netflix:latest' } }
    }
}
```

Phase 4: Monitoring
1. Install Prometheus

    Download and configure Prometheus, then expose it on port 9090.

2. Install Grafana

    Set up Grafana and add Prometheus as a data source.
    Access Grafana at http://<your-server-ip>:3000.

Phase 5: Notifications
1. Email Notifications Setup in Jenkins

    Configure Jenkins to send email notifications or integrate it with other notification services like Slack.

Phase 6: Kubernetes Deployment
1. Create a Kubernetes Cluster and Monitor

    Set up a Kubernetes cluster with node groups.
    Install Prometheus and Node Exporter using Helm to monitor the cluster.

2. Deploy with ArgoCD

    Install ArgoCD and use it to deploy the Netflix app on the Kubernetes cluster.

Conclusion

This project demonstrates how to build, deploy, secure, and monitor an application using a modern DevSecOps pipeline. It leverages tools such as Jenkins for CI/CD, SonarQube and Trivy for security checks, Docker for containerization, and Kubernetes for deployment and scaling.

Feel free to clone the repository and follow the steps outlined here to set up your own DevSecOps pipeline.
License

This project is licensed under the MIT License.
