## Project Description

This architecture shows a **CI/CD pipeline** that automates the process of building, testing, scanning, and deploying Board Game Database Full-Stack Web Application. This web application displays lists of board games and their reviews. While anyone can view the board game lists and reviews, they are required to log in to add/ edit the board games and their reviews. The 'users' have the authority to add board games to the list and add reviews, and the 'managers' have the authority to edit/ delete the reviews on top of the authorities of users. using tools like **Jenkins**, **Docker**,  **SonarQube**, **Trivy**, and **Kubernetes**.

## 🚀 CI/CD Pipeline Overview

The following tools are used to automate the CI/CD process:

| Stage                | Tool(s) Used       |
|---------------------|--------------------|
| Source Control       | GitHub             |
| Continuous Integration | Jenkins           |
| Dependency Management | Maven             |
| Static Code Analysis | Trivy, SonarQube   |
| Build & Image Creation | Docker           |
| Container Image Scanning | Trivy         |
| Artifact Registry    | Docker Hub         |
| Deployment           | Kubernetes (k3s)   |

---

## ⚙️ Pipeline Stages

1. **Git Checkout**  
   Pull the latest code from GitHub.

2. **Compilation & Testing**  
   Use Maven to compile and run tests.

3. **File System Vulnerability Scan**  
   Use Trivy to scan local source code for vulnerabilities.

4. **SonarQube Analysis**  
   Run static analysis for code quality and bugs.

5. **Quality Gate**  
   Ensure the SonarQube quality gate passes before proceeding.

6. **Build & Package**  
   Package the application into a `.jar` file.

7. **Docker Image Build & Tag**  
   Build and tag the Docker image.

8. **Docker Image Vulnerability Scan**  
   Use Trivy to scan the image for known vulnerabilities.

9. **Push Docker Image**  
   Push the image to Docker Hub.

10. **Deploy to Kubernetes (k3s)**  
    Deploy using `kubectl apply`.

11. **Deployment Verification**  
    Confirm pod and service creation.

---

## 🧪 Technologies Used

- **Java 17**
- **Maven**
- **Spring Boot**
- **Jenkins**
- **Docker**
- **Trivy**
- **SonarQube**
- **Kubernetes (k3s)**
- **GitHub**

---

## 🖥️ Local Setup

> The following tools must be installed and configured locally:

1. **Jenkins**  
   - Install Jenkins: [https://www.jenkins.io/download/](https://www.jenkins.io/download/)
   - Install required plugins:
     - Docker
     - Kubernetes CLI
     - SonarQube Scanner
     - Maven Integration
     - GitHub Integration

2. **Docker**  
   - Install Docker: [https://docs.docker.com/get-docker/](https://docs.docker.com/get-docker/)

3. **SonarQube**  
   - **Start SonarQube using Docker with the following command:**  
   ```bash
   docker run -d --name SonarQube -p 9000:9000 SonarQube:lts
   ```
   - Access SonarQube at **http://\<your-server-ip\>:9000**.
   - Configure token and project key in Jenkins

4. **Trivy**  
``` bash
cat << EOF | sudo tee -a /etc/yum.repos.d/trivy.repo
[trivy]
name=Trivy repository
baseurl=https://aquasecurity.github.io/trivy-repo/rpm/releases/\$basearch/
gpgcheck=1
enabled=1
gpgkey=https://aquasecurity.github.io/trivy-repo/rpm/public.key
EOF
sudo yum -y update
sudo yum -y install trivy
```
5. **K3s (Kubernetes)**  
   - sh ```curl -sfL https://get.k3s.io | sh - ```
   - Get `kubeconfig` and configure Jenkins with it

---

## 🧩 Jenkins Pipeline

The Jenkins pipeline performs all CI/CD steps automatically. Here's the key configuration:

```groovy
pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven'
    }

    environment {
        SCANNER_HOME = tool 'sonnar-scanner'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'mygitcredentials', url: 'https://github.com/SherinHammad/CICD-Pipeline-Boardgame.git'
            }
        }

        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }

        stage('Test') {
            steps {
                sh "mvn test"
            }
        }

        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_AUTH_TOKEN')]) {
                        sh '''
                            $SCANNER_HOME/bin/sonar-scanner \
                            -Dsonar.projectName=BoardGame \
                            -Dsonar.projectKey=BoardGame \
                            -Dsonar.java.binaries=. \
                            -Dsonar.login=$SONAR_AUTH_TOKEN
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    timeout(time: 2, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: false, credentialsId: 'sonarqube-token'
                    }
                }
            }
        }

        stage('Build') {
            steps {
                sh "mvn package"
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
               script {
                   withDockerRegistry(credentialsId: 'mydockertoken', toolName: 'docker') {
                        sh "docker build -t sherinhammad/boardgame:latest ."
                   }
               }
            }
        }

        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html sherinhammad/boardgame:latest"
            }
        }

        stage('Push Docker Image') {
            steps {
               script {
                   withDockerRegistry(credentialsId: 'mydockertoken', toolName: 'docker') {
                        sh "docker push sherinhammad/boardgame:latest"
                   }
               }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(credentialsId: 'k3s-kubeconfig') {
                    sh 'kubectl apply -f deployment-service.yaml'
                }
            }
        }

        stage('Verify the Deployment') {
            steps {
                withKubeConfig(credentialsId: 'k3s-kubeconfig') {
                    sh 'kubectl get pods'
                    sh 'kubectl get svc'
                }
            }
        }
    }
}
